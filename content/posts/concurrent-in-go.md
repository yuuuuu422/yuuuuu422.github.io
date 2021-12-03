---
title: go并发的几个问题
date: 2021-12-02T23:30:20+08:00
Toc: On
---

## 前言

作为世界上除了PHP之外最好的语言golang，只需`go`关键字修饰函数，就可以直接启动一个goroutine(协程)运行，但在实际的场景中，我们需要考虑到协程的数量，其之间的同步与通信，以及精确控制其结束。

## 如何控制协程的通信

### 引入全局变量

这是最简单也是最容易想到的方法：虽然goroutine的退出只能由其自身的决定，不允许从外部直接控制，不过我们可以通过引入全局变量，所有的goroutine都共享这个变量，并且不断寻查其是否更新，在主程序中对其更改，goroutine勘测到其变化后做出反应。

```go
package main
import (
	"fmt"
	"time"
)
var running bool
func run()  {
	for running  {
		fmt.Println("running")
		time.Sleep(500*time.Millisecond)
	}
	fmt.Println("stop now")
}

func main(){
	running=true
	go run()
	go run()
	time.Sleep(time.Second)
	running=false
	time.Sleep(time.Second)
}

/*
out:
running
running
running
running
stop now
stop now
*/
```

这种写法看似很简单，但是还是有好几个问题：

1. 全局变量存在数据同步问题，如果有多个写入需要加锁处理。
2. 协程之间的通信量很小，只有事先定义的全局变量，并且只能单向从主程序通知给协程。

### 利用channel通信

相信写go的兄弟，一定对这一句话不陌生：

>Go语言的并发模型是CSP（Communicating Sequential Processes）通信顺序进程，提倡通过通信共享内存而不是通过共享内存而实现通信。

这里简单谈谈我的理解:

共享内存是什么?如果在一个系统中，不同进程或者线程共享一块内存，那么他们之间不需要进行平凡的交互，如果有大量的数据传输，也省去了数据拷贝的消耗。

但是这有一个很大的问题，就是多线程下，共享一块内存，肯定会存在数据冲突。为了对抗这种冲突，人们发明了很多机制，比如加锁，信号量，各种调度算法等等，但是这毫无都会对并发的性能造成影响。(但并不是说全部都不行，比如[深度 | 字节跳动微服务架构体系演进](https://zhuanlan.zhihu.com/p/382833278)）

最终“通过通信来实现进程/线程间交互”的方案脱颖而出,go就在语言层提供了channel来实现这一方案，简单理解就是设计的时候，对于消息队列，只提供读写接口，而对于内部的实现你完全不用去在意，看起来消息队列就像是共享内存一样了。然而你的消息队列可以利用socket进行通信。

通过看channel的源码，可以看出它其实就是一个队列加一个轻量锁

```go
type hchan struct {
   qcount   uint           // total data in the queue
   dataqsiz uint           // size of the circular queue
   buf      unsafe.Pointer // points to an array of dataqsiz elements
   elemsize uint16
   closed   uint32
   elemtype *_type // element type
   sendx    uint   // send index
   recvx    uint   // receive index
   recvq    waitq  // list of recv waiters
   sendq    waitq  // list of send waiters

   // lock protects all fields in hchan, as well as several
   // fields in sudogs blocked on this channel.
   //
   // Do not change another G's status while holding this lock
   // (in particular, do not ready a G), as this can deadlock
   // with stack shrinking.
   lock mutex
}
```

再谈谈select机制，可以理解为select, poll, epoll 相似的功能：监听多个描述符的读/写等事件，属于基于事件的并发处理(欸好像和之前看csapp第12章的知识连起来了)，简单来说就是监听多个channel，每一个case都是一个事件，按照先后(如果相同则随机)执行，如果没监听的事件暂时堵塞则会执行default。

```go
package main

import (
   "fmt"
   "time"
)

func main() {
   output1 := make(chan string, 5)
   go write(output1)
   for s := range output1 {
      fmt.Println("res:", s)
      time.Sleep(time.Second*2)
   }
}

func write(ch chan string) {
   for {
      select {
      case ch <- "hello":
         fmt.Println("write hello")
      default:
         fmt.Println("channel full")
      }
      time.Sleep(time.Millisecond * 500)
   }
}
/*
write hello
res: hello
write hello
write hello
write hello
res: hello
write hello
write hello
write hello
channel full
......
*/
```

## 控制并发量

准备写这里的时候，在知乎上看到一个老哥说可以通过`runtime.GOMAXPROCS(n)`直接修改最大线程数... 

这是对并发和并行没有弄清楚

```
多线程程序在一个核的cpu上运行，就是并发。
多线程程序在多个核的cpu上运行，就是并行。
```

当一个函数创建为goroutine时，编译器会将其视为一个独立的工作单元。这个单元会被调度到**可用的逻辑处理器**（可用的核数）上执行。线程是和逻辑处理器绑定的。而`runtime.GOMAXPROCS(n)`就是分配n个逻辑处理器。但我们这里谈并发，还是在一个偏~~微观~~的层面，可以说这个回答是毫无相关了。

我们首先看看过高的并发会导致什么问题：

```go
package main

import (
	"fmt"
	"math"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	for i := 0; i < math.MaxInt32; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			fmt.Println(i)
			time.Sleep(time.Second)
		}(i)
	}
	wg.Wait()
}
```

```
panic: too many concurrent operations on a single file or socket (max 1048575)

goroutine 1127972 [running]:
internal/poll.(*fdMutex).rwlock(0xc000110280, 0x113500, 0x7600000001)
        D:/go/src/internal/poll/fd_mutex.go:147 +0x146
internal/poll.(*FD).writeLock(...)
        D:/go/src/internal/poll/fd_mutex.go:239
internal/poll.(*FD).Write(0xc000110280, 0xc17470e5f0, 0x8, 0x8, 0x0, 0x0, 0x0)
```

报错是由`fmt.println`引起的，对单个 file/socket 的并发操作个数超过了系统上限，那如果我们把`fmt.println`换成并发安全的`log.println`呢？

运行后，goland直接退出，chrome浏览器也闪退。每个协程至少需要消耗 2KB 的空间，在骤减的内存空间下，程序运行很容易崩溃，总而言之就是并发的控制不当导致系统的资源被耗尽了。

不同的应用程序对资源的需求是不同的，比如如果是并发对本地资源的操作，那么应该需要考虑系统资源的承受能力；如果是对外端口扫描、密码破解，那还需要考虑会不会触发风控警告等等。总之，并发的上限应该由程序主动控制。

```go
package main

import (
   "log"
   "sync"
   "time"
)
func crack(taskChan chan int,wg *sync.WaitGroup){
   for task:=range taskChan{
      log.Println("crack: ",task)
      time.Sleep(time.Second)
      wg.Done()
   }
}
func main() {
   var wg sync.WaitGroup
   threat:=10
   taskChan:=make(chan int,threat)
   for i:=0;i<threat;i++{
      go crack(taskChan,&wg)
   }

   for i:=0;i<100;i++{
      wg.Add(1)
      taskChan<-i
   }
   wg.Wait()
   close(taskChan)
}
```

上面这个实例很好理解，相当于创建了10个并发的crack消费者，range感知taskChan的变化，再通过一个for依次把目标输送给goroutine。

实际上，除了控制并发之外，有时候我们还需要控制发包的速率，避免过快触发警告，可以利用`time.NewTicker(rateLimit)`计时器来控制发包

```go
package main

import (
	"log"
	"time"
)

func main(){
  rate:=10
  rateLimit:=time.Second/time.Duration(rate)
  ticker := time.NewTicker(rateLimit)
  worker:= func() {
    for {
      <-ticker.C
      log.Println("ok")
    }
  }
  go worker()
  go worker()
  time.Sleep(time.Second*10)
}
```

但如果在实际工程的时候，需要考虑一些问题。比如如果是多ip的扫描，应该给每个ip分发一个ticker而不是共享，不然对效率会有比较大的损失。

## 退出协程的几种方式

关于协程，我们不仅要关注创建和通信，还要关注如何合理的退出。当然之前说到全局变量的确可以，但是不推荐，以下讲述三种方式退出协程。

### for-range退出

之前说过range可以感知channel的变化，如果协程只从一个channel中读取数据，那么下列的程序即可主动退出协程

```go
func main(){
   channel:=make (chan int)
   go func() {
      defer fmt.Println("exit")
      for x:=range channel{
         log.Println(x)
      }
   }()

   for i:=1;i<=10;i++{
      channel<-i
      if i==5{
         close(channel)
         break
      }
   }
   time.Sleep(time.Second)
}
```

### select退出

上述只是针对单个channel的读取，select的多路复用可以处理多个chanel，但是其并不能感知channel的关闭，会一直读取到0值。因为关闭的channel可以读取，但是写入会引发panic。不过我们可以用`,ok`来解决这个问题。

```go
go func() {
		defer log.Println("exit")
		for {
			select {
			case x,ok:=<-in:
				if !ok {
					return
				}
				log.Println("continue",x)
			case <-other:
				log.Println("continue")
			}
		}
	}()
```

上述的例子只要channel in关闭则会主动退出协程。但还是存在多个channel，如果有指定个channel退出，则退出协程的情况，这里要用到**select不会在nil的通道上进行等待**，所以我们可以把关闭的通道全部设置为nil，在循环底部加上判断即可。

```go
go func() {
   for {
      select {
      case x, ok := <-in1:
         if !ok {
            in1 = nil
         }
      case y, ok := <-in2:
         if !ok {
            in2 = nil
         }
      if in1 == nil && in2 == nil {
         return
      }
   }
}()
```

### 使用专门通道退出协程

这里传入了一个专门的channel`stopCh`,当main函数执行close(stopCh)时，所有协程里的`case <-stopCh`都会收到信号，进而关闭，这比给stopCh发送多个数据方便多了

```go
func worker(stopCh <-chan struct{}) {
   go func() {
      defer fmt.Println("worker exit")
      // Using stop channel explicit exit
      for {
         select {
         case <-stopCh:
            log.Println("Recv stop signal")
            return
         default:
            log.Println("running")
            time.Sleep(time.Second)
         }
      }
   }()
   return
}
func main(){
   stopCh:=make(chan struct{})
   go worker(stopCh)
   go worker(stopCh)
   go worker(stopCh)
   time.Sleep((time.Second)*2)
   close(stopCh)
   time.Sleep(time.Second)
}
```

## Reference

- https://www.topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/channel.html
- https://studygolang.com/articles/16774

## 写在最后

说一个很有意思的事情，笔者曾在去年寒假认认真真学了两个月go，原因呢，主要还是想要~~专精~~于一门语言吧。c++大一留下了很不好的印象，php动态类型不太能接受，最后选择了golang。学习路线大概是：

1. [Go语言圣经](https://books.studygolang.com/gopl-zh/) 这本书的评价相当高，我也首先选择了这本，大概在是异常的时候放弃了，感觉这本书的例子很有高度，但不太适合初学者，更像是有一定经验的gopher日常回味的感觉。

2. [the way to go ](https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/directory.md)后来加了一个go语言学习群，在里面有师傅推荐了这一本书，然后就顺着一点一点看，看到并发那一章的时候，卡住了...可能是思想上没能转变过来，最后无意间搜到了一本非常通俗易懂的书

3. [Go语言中文文档](https://www.topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/%E5%B9%B6%E5%8F%91%E4%BB%8B%E7%BB%8D.html) 准确来说这并不是一本书，是一个叫枯藤的go语言爱好者结合前人的资料，总结下来的一份非常全面的文档，后续的学习也基本上是在这个的基础上，不过寒假的学习基本上到gin就结束了，rpc什么的都是后续回学校有的没的看一些。还有收集一些非常好的资料，但是都甚至没能深入看看。

4. [Go语言高级编程](https://chai2010.cn/advanced-go-programming-book/) 这本书的需要一定的基础，从目录->`CGO`,`汇编`,`RPC`等等也能看出来

5. [build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang) 主要是web方面，也是评价很高

6.  [Go 语言设计与实现](https://draveness.me/golang/) 刚刚点开的时候发现出书了！！！必须支持！！信仰师傅是某天操作系统课上，骏哥推给我的。如果真要对标一本其他的书的话，这本书在go上的定位可能和《深入了解java虚拟机》在java上一样。(不过我只看了基础知识和编译原理部分)

![contents-mindnode](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211203181902.png)

但是我想说的是什么呢，之前和骏哥聊天，说想学一门语言。我的习惯是在知乎，豆瓣看各种推荐，书评，然后罗列一大堆，再**精挑细选**一本慢慢看。骏哥呢？一个字，`调`。go？并发好像是优势，直接上手写，不会的就看官方文档。java？直接编译jdk，开调。solidity？编译evm虚拟机，开调。

这就导致了一点，我好像永远停留在语言的层面上，并为之此乐此不疲，但也只是一些皮毛功夫。语言只是工具，项目驱动学习效果会更好一些。比如学习springboot，比起上来就依赖注入，控制反转等概念的介绍，不如先抄或者直接照搬一个别人的代码跑起来，断点看看数据的流向，有问题再逐个学习。

这样来看，新人学习的确很容易进入一个误区，就是想办法让自己学的全面，各种铺路，实话说到现在我也还没能改掉这个毛病。我们得明白学这门语言是为了什么，大多时候毫无意义的准备都是因为迷茫，如果你是为了想写扫描器学go，那不如了解一些基础语法后，马上上手项目。我感觉这是有一些本末倒置了,书籍还是适合在有一些经验的基础上，作为一种内功提升的工具，让你看完后感觉：`居然还能这样?我之前的写法真是nt`。该踩的坑还是要踩的，学习之路漫漫无期...
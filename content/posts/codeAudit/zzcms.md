---
title: "zzcms code Audit"
date: 2021-05-19T19:25:18+08:00
toc : on
---
## 准备工作 

虽然事先准备审这个cms,但是找源码还是花了不少时间，官网只有最新的版本，这里如果大家想要复现就直接下载这里的就好了。

为了模拟真实环境，我们在**phpstudy**上配置一下站点,并修改hosts文件即可开始动手啦。

## 0x01 sql注入

进入界面,就有一个很明显的搜索框,我们尝试`1' or 1=1 #`

![image-20210520195446877](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520195446877.png)

发现单引号被转义，`search.php`包含了`/inc/conn.php`,而后者又包含了`inc/stopsqlin.php`，对应代码:

![image-20210520200128259](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520200128259.png)

可以看到这里会对我们`POST`,`GET`以及`COOKIE`中的数据进行转义,那如果想要继续进行下去我们要么就是找到没有包含 `/inc/conn.php`的组件，要么就是找到拼接sql语句的地方。

`user/check.php`是对我们身份进行验证的文件，如果cookie设置有`username`和`password`，就会执行一次sql查询,并且该文件奇幻的是没有包含上面说的过滤文件，还自己写了一个过滤函数，还只对`username`进行过滤,那么我们尝试一下password处。	

![image-20210520201929542](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520201929542.png)

不过这个路由设有对验证码的识别，而`user/index.php`包含了该函数,那我们可以直接在`index.php`测试。

![image-20210520203435308](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520203435308.png)

那么这里也理所当然可以通过盲注拿数据,不过我在本地搭建的环境比较慢，平均延迟在4秒左右，就没有跑了，师傅们见谅。

比较遗憾的是管理员路由设置的为**session**参数验证，无法通过万能密码的方法登陆。

## 0x02 任意文件删除

在`user/adv.php`处，

![image-20210520205041873](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520205041873.png)

不得不说这种对`$_REQUEST`没有限制的代码真的很恐怖,逻辑其实很简单只要`$action`等于**modify**和`$img`不等于`$oldimg`即可，并且这里没有对``$oldimg``过滤导致我们可以直接把`../../admin/admin.php`删除。

## 0x03 网站重装

在Install目录下，step1.php会首先判断是否存在install.lock文件，但是在step2.php及之后的文件都没有对其进行判断,并且step可控。

![image-20210520205732410](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520205732410.png)

![image-20210520210143862](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520210143862.png)

## 0x04 任意密码修改

漏洞点在`one/getpassword.php`

```php
elseif($action=="step3" && @$_SESSION['username']!=''){
$passwordtrue = isset($_POST['password'])?$_POST['password']:"";
$password=md5(trim($passwordtrue));
query("update zzcms_user set password='$password',passwordtrue='$passwordtrue' where username='".@$_SESSION['username']."'");
```

利用样式和网址重装有些相似，也是POST**action**到step3直接修改密码,但这里需要我们知道被修改用户的`$_SESSION['username']`,而在step1中我们可以看到：

```php
if ($action=="step1"){
$username = isset($_POST['username'])?$_POST['username']:"";
$_SESSION['username']=$username;
......
```

`$_SESSION['username']`有一个被我们赋值的过程，那我们只需先在step1中输入需要修改的username获取session，就可直接跳到step3修改密码。步骤如下：

![image-20210520221904775](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520221904775.png)

然后直接修改密码,把action改为step3即可。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520222027782.png)

## 0x05 反射型xss

在`inc/top.php`处没有包含`inc/conn.php`，我们只需要将标签闭合即可实现反射型xss。

![image-20210520210717678](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520210717678.png)

![image-20210520212245204](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520212245204.png)

事实上`admin`用户的后台管理基本上没有什么过滤，存在很多存储型xss,但归于业务原因, 拿到admin权限也就不在乎这个级别漏洞了,感兴趣的师傅可以自行复现。

## 0x06 上传webshell

在`uploadimg_form.php`可以文件上传，不过这个路由非常突兀让我有了一种好像做ctf题的感觉..

回到正题,我们看看后端是怎么处理我们的文件的:`uploadimg.php`

```php
function upfile() {
//是否存在文件
if (!is_uploaded_file(@$this->fileName[tmp_name])){
   echo "<script>alert('请点击“浏览”，先选择您要上传的文件！\\n\\n支持的图片类型为：jpg,gif,png,bmp');parent.window.close();</script>"; exit;
}
//检查文件大小
if ($this->max_file_size*1024 < $this->fileName["size"]){
   echo "<script>alert('文件大小超过了限制！最大只能上传 ".$this->max_file_size." K的文件');parent.window.close();</script>";exit;
}
//检查文件类型
if (!in_array($this->fileName["type"], $this->uptypes)) {
   echo "<script>alert('文件类型错误，支持的图片类型为：jpg,gif,png,bmp');parent.window.close();</script>";exit;
}
//检查文件后缀
$hzm=strtolower(substr($this->fileName["name"],strpos($this->fileName["name"],".")));//获取.后面的后缀，如可获取到.php.gif
if (strpos($hzm,"php")!==false || strpos($hzm,"asp")!==false ||strpos($hzm,"jsp")!==false){
echo "<script>alert('".$hzm."，这种文件不允许上传');parent.window.close();</script>";exit;
}
```

首先判断是否存在文件,再判断文件大小，然后判断文件类型,这里用**GIF89a**可以绕过，最后是一个黑名单的后缀验证,我们使用phtml即可绕过。

![image-20210520214742983](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520214742983.png)

需要注意的是，`phtml`需要管理员在apche配置中设置,不然会直接把源码打印没有解析,并且我在测试的过程中发现php7即使设置了解析,访问`.phtml`文件的效果是直接下载文件,而php5版本可以成功。

![image-20210520220014941](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-20-image-20210520220014941.png)

## 总结

虽然是入门级别的框架，但也来来回回审了两三天，不得不说代码审计一定需要静下心来看慢慢看。这个框架的漏洞利用点更多是在设计者本身对输入的过滤不严格所导致的，而且逻辑上也有很大的问题。之前在师傅博客看到这样一句话：**知识面宽度决定攻击面广度,知识链深度决定攻击链的长度**，虽然这是一次简单的白盒审计，不过也给黑盒测试提供了很多思路。

不过也是这次审计让我意识到自己对代码的敏感能力还是不够高，之前复现一些laravel和Yii的大型框架基本上都是拿着大师傅们的脚本直接打。学习还是一步一步好,戒骄戒躁。


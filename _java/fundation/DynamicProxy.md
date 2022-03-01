---
title: DynamicProxy
---

## 前沿

代理模式Java当中最常用的设计模式之一。其特征是代理类与委托类有同样的接口，代理类主要负责为委托类预处理消息、过滤消息、把消息转发给委托类，以及事后处理消息等。

>举个例子 , 存在一个 对象A , 但是开发人员不希望程序直接访问 对象A , 而是通过访问一个中介对象B来间接访问 对象A , 以达成访问对象A的目的 . 此时 , 对象A 被称为 "委托类" , 对象B 被称为 "代理类" .

总的来说 , 使用代理模式有如下两个优点 :

1. 可以隐藏委托类的实现.
2. 可以实现客户与委托类间的解耦 , 在不修改原目标对象的前提下 , 提供额外的功能操作 , 扩展目标对象的功能.

如果**根据字节码的创建时机**来分类，可以分为静态代理和动态代理：

- 所谓**静态**也就是在**程序运行前**就已经存在代理类的**字节码文件**，代理类和真实主题角色的关系在运行前就确定了。
- 而动态代理的源码是在程序运行期间由**JVM**根据反射等机制**动态的生成**，所以在运行前并不存在代理类的字节码文件

## 静态代理

这里通过一个例子先学习静态代理。

定义一个接口：

```java
package com.theoyu.proxy.staticproxy;

public interface Rental {
    void sale();
}
```

委托类，继承接口并实现接口的方法：

```java
package com.theoyu.proxy.staticproxy;

public class Entrust implements Rental {

    @Override
    public void sale() {
        System.out.println("Rental house");
    }
}
```

代理类，如果我们想对方法的实现进行修改，可以不用直接修改委托类而是通过代理中介的方式转达

```java
package com.theoyu.proxy.staticproxy;


public class StaticAgent implements Rental {
    private Rental target;
    public StaticAgent(Rental target){
        this.target=target;
    }
    @Override
    public void sale(){
        System.out.println("The price of renting a house is between 1K and 3K"); // 增加新的操作
        target.sale(); // 调用Entrust委托类的sale方法
    }
}
```

测试类，生成委托类对象，并分别通过委托和代理方式执行方法

```java
package com.theoyu.proxy.staticproxy;

public class StaticTest {
    // 静态代理使用示例
    public static void consumer(Rental subject) {
        subject.sale();
    }
    public static void main(String[] args) {
        Rental test = new Entrust();
        System.out.println("---使用代理之前---");
        consumer(test);
        System.out.println("---使用代理之后---");
        consumer(new StaticAgent(test));
    }
}
```

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/09/20210925010228.png)

编译后，在out文件夹下生成了代理类，同时我们没有修改源代码(委托类)就达到了目的。

虽然静态代理实现简单，且不侵入原代码，但是，当场景稍微复杂一些的时候，静态代理的缺点也会暴露出来。

1、 当需要代理多个类的时候，由于代理对象要实现与目标对象一致的接口，有两种方式：

- 只维护一个代理类，由这个代理类实现多个接口，但是这样就导致**代理类过于庞大**
- 新建多个代理类，每个目标对象对应一个代理类，但是这样会**产生过多的代理类**

2、 当接口需要增加、删除、修改方法的时候，目标对象与代理类都要同时修改，**不易维护**。

这就需要我们用到动态代理的知识，动态代理类的源码是在程序运行期间由**JVM**根据反射等机制动态的生成，所以不存在代理类的字节码文件。代理类和委托类的关系是在程序运行时确定。下面我们以JDK动态代理为例：

## 动态代理

JDK动态代理核心为以下两个类

- `java.lang.reflect.InvocationHandler`

- `java.lang.reflect.Proxy`

`java.lang.reflect.InvocationHandler` 为调用处理器接口 , **该接口定义了一个 `invoke()` 方法 , 用于集中处理在动态代理类对象上的方法调用 . 当程序通过代理对象调用某一个方法时 , 该方法调用会被自动转发到 `InvocationHandler.invoke()` 方法来进行调用 .**

```java
package java.lang.reflect;
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

> proxy: 代理类实例对象
>
> method: 被调用方法名 , 该参数的类型为 `java.lang.reflect.Method` 类型
>
> args : 被调用方法的参数数组

继承 `java.lang.reflect.InvocationHandler` 接口，通过实现 `invoke()` 方法来添加代理访问的逻辑 . 逻辑代码块中除了调用委托类的方法 , 还可以添加自定义逻辑 ；AOP( 面向切面编程 )就是这样实现的。

`java.lang.reflect.Proxy`是 Java 动态代理机制生成的所有动态代理类的父类，它提供了一组静态方法来为一组接口动态地生成代理类及其对象。 

```java
// 方法 1: 该方法用于获取指定代理对象所关联的调用处理器  
static InvocationHandler getInvocationHandler(Object proxy)   
  
// 方法 2：该方法用于获取关联于指定类装载器和一组接口的动态代理类的类对象  
static Class getProxyClass(ClassLoader loader, Class[] interfaces)   
  
// 方法 3：该方法用于判断指定类对象是否是一个动态代理类  
static boolean isProxyClass(Class cl)   
  
// 方法 4：该方法用于为指定类装载器、一组接口及调用处理器生成动态代理类实例  
static Object newProxyInstance(ClassLoader loader, Class[] interfaces, InvocationHandler h)  
```

JDK 原生动态代理的执行流程分为如下三步.

1. 通过实现 `java.lang.reflect.InvocationHandler` 接口来创建自定义的调用处理器(`InvocationHandler`) .
2. 为 `java.lang.reflect.Proxy` 类指定一个类加载器(`ClassLoader`) , 一组接口(`Interfaces`) 和 一个调用处理器(`InvocationHandler`)
3. 调用 `java.lang.reflect.Proxy.newProxyInstance()`方法 , 分别传入类加载器 , 被代理接口 , 调用处理器 ; 创建动态代理实例对象.

这里的例子接口和委托类保持不变(需要改对于包名)，仅修改代理类和测试类。

代理类：

```java
package com.theoyu.proxy.dynamicproxy;
import java.lang.reflect.Method;
import java.lang.reflect.InvocationHandler;
public class DynamicAgent implements InvocationHandler {
    //target为委托类对象
    private Object target;
    public DynamicAgent(Object target){
        this.target=target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("The price of renting a house is between 1K and 3K");
        Object result = method.invoke(target,args);
        return result;
    }
}

```

测试类：

```java
package com.theoyu.proxy.dynamicproxy;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.net.ProxySelector;

public class DynamicTest {
    public static void main(String[] args) {
        Entrust testEntrust = new Entrust();
        // 获取CLassLoader
        ClassLoader classLoader = testEntrust .getClass().getClassLoader();
        // 获取所有接口
        Class[] interfaces = testEntrust .getClass().getInterfaces();
        // 获取一个调用处理器
        InvocationHandler invocationHandler = new DynamicAgent(testEntrust);
        // 查看生成的代理类
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        // 创建代理对象
        Rental proxy = (Rental) Proxy.newProxyInstance(classLoader,interfaces,invocationHandler);
        // 判断对象是否为代理类
        System.out.println(Proxy.isProxyClass(proxy.getClass()));
        // 调用代理对象的sayHello()方法
        proxy.sale();
    }
}
```

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/09/20210925202751.png)

调试发现，的确是在`proxy.sale()`调用时，转发到调用处理器的`invoke()`方法。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/09/20210925203633.png)

我们在编译后，没有发现代理类有关的class文件，这就是动态代理和静态代理不同的地方，不难发现在main函数中 有这样一句`System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");`,这个的作用是调用动态代理时在程序根目录生成**`com.sun.proxy.$Proxy0.class`** 文件 , 它就是动态代理类的字节码文件 .

关于proxy的具体实现源码就不过多阐述了，后续学习碰面再细说。


---
title: Reflection
key: Reflection
---

## 前言

@[JAVACORE](https://dunwu.github.io/javacore/)

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210820124011.png)

反射就是在运行时才知道要操作的类是什么，并且可以在运行时获取到任何类的成员方法(Methods)、成员变量(Fields)、构造方法(Constructors)等信息，还可以动态创建Java类实例、调用任意的类方法、修改任意的类成员变量值等。实际上在类加载器一节中我们初始化CustomClassLoader实例的时候就用到了反射去获取Methods。

## 概述

java反射流程图@[**小阳**](https://xz.aliyun.com/u/24696) 

![image-20220116212904773](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/01/20220116212904.png)

之前的编译加载在ClassLoader一节基本上都介绍过，这里需要注意的是在生成class文件后，也就生成了Class对象，里面拥有其获取成员变量Field，成员方法Method和构造方法Constructor等方法，注意这里的**Class**是大写：

>Class是一个实实在在的类,就如同我们自己定义的类一样，在java.lang.Class下，Class对象也就是这个Class类的实例

所以反射实际上就是获取一个类的Class对象，然后在用Class对象中的获取成员变量Field，成员方法Method和构造方法Constructor等方法，再去动态获取一个类或者调用一个类的属性，变量，构造方法等方式。

```java
private Class(ClassLoader loader) {
    // Initialize final field for classLoader.  The initialization value of non-null
    // prevents future JIT optimizations from assuming this final field is null.
    classLoader = loader;
}
```

因为Class类中，构造器是私有的，所以我们无法通过

```java
class person = new Class();
```

直接获取Class对象。

## 获取Class对象

有四种方式获取Class对象：

- `Class c1 = com.theoyu.reflection.DemoReflection.class;`

  需要导入类的包

- ```java
  DemoReflection demo = new DemoReflection();
  Class c3 = demo.getClass();
  ```

  需要创建一个对象，不具有使用反射的机制意义。

- `Class c3 = Class.forName("com.theoyu.reflection.DemoReflection");`

- `Class c4 = ClassLoader.getSystemClassLoader().loadClass("com.theoyu.reflection.DemoReflection")`

其中第三种和第四种都利用了反射获取Class对象，但这两者是有区别的。loadClass相对于Class.fromName更加底层，

在上一节我们知道类的加载有三个过程

- 加载
- 链接(验证，准备，解析)
- 初始化

在forName中有一个默认为true的参数，表示装载类的时候是否初始化该类，即调用类的静态块的语句及初始化静态成员变量。而loadClass中`resolveClass`方法是链接类,默认为false，所以不会走到初始化这步，需要自己初始化实例。

## 反射创建类对象

就拿上一节初始化testClass为例：

```java
Class <?> testClass =customClassLoader.loadClass(className);
System.out.println("[+]class:"+testClass);
Object testInstance=testClass.newInstance();
Method method=testInstance.getClass().getMethod("hello");
String string= (String) method.invoke(testInstance);

```

- 获取Class对象：loadClass，forName
- 实例化类对象的方法：newInstance
- 获取函数的方法：getMethod
- 执行函数的方法：invoke

invoke方法位于java.lang.reflect.Method类中，用于执行某个的对象的目标方法

```java
public Object invoke(Object obj, Object... args)
```

11月10日补充 :

看了看CC链的反射，忽然发现他的第一个参数和预想不太一样，果然是之前错过了一些细节。

>If the underlying method is static, then the specified obj argument is ignored. It may be null.

实际上通过invoke获取方法如果是静态方法的话，本身就可以直接通过类名来访问，不需要实例，那么第一个参数完全可以是为空的。

拿到object对象：

```java
 Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(null);
```



## 反射构造Runtime执行

```java
package com.theoyu.reflection;

import java.lang.reflect.Method;

public class ReflectDemo {
    public static void main(String[] args) {
        try {
            Class c1 = Class.forName("java.lang.Runtime");
           Object  object =  c1.newInstance();
           Method method = c1.getMethod("exec", String.class);
           method.invoke(object,"/System/Applications/Calculator.app/Contents/MacOS/Calculator");

        } catch (Exception e){
           e.printStackTrace();
        }
    }
}
```

 运行发现报错：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210823000910.png)

发现**Runtime**类的构造方法是私有的，可是为什么设计者会把一个默认构造函数设置为私有的呢？我对java的设计模式并不是很了解，这里引用**Phith0n**师傅的解释：

>这是一种常见的设计模式:“单例模式”，有时也称为工厂模式。
>
>比如，对于Web应用来说，数据库连接只需要建立一次，而不是每次用到数据库的时候再新建立一个连接，此时作为开发者你就可以将数据库连接使用的类的构造函数设置为私有，然后编写一个静态方法来获取。
>
>这样，只有类初始化的时候会执行一次构造函数，后面只能通过 getInstance 获取这个对象，避免建 立多个数据库连接。

```java
public class TrainDB {
    private static TrainDB instance = new TrainDB();
    public static TrainDB getInstance() {
        return instance;
		}
		private TrainDB() { // 建立连接的代码...
		} 
}
```

我们回头看看**Runtime**的源码，果然和上面数据库连接的例子如出一辙，并且这里提供了`getRuntime`方法去

![image-20210924213545164](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/09/20210924213547.png)

我们只需要通过`Runtime.getRuntime()`获取Runtime对象即可。

```java
package com.theoyu.reflection;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

public class ReflectDemo {
    public static void main(String[] args) {
        try {
            Class c1 = Class.forName("java.lang.Runtime");
           Method method = c1.getMethod("exec", String.class);
            Object object = c1.getMethod("getRuntime").invoke(c1);           		method.invoke(object,"/System/Applications/Calculator.app/Contents/MacOS/Calculator");

        } catch (Exception e){
           e.printStackTrace();
        }
    }
}

```

那我们有没有可能通过反射直接访问私有的方法或者变量呢？

答案是可以的,通过`getDeclaredConstructor()`拿到私有的构造方法，再通`Constructor/Field/Method.setAccessible(true)`绕过访问限制。

```java
package com.theoyu.reflection;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

public class ReflectDemo {
    public static void main(String[] args) {
        try {
          Class c1 = Class.forName("java.lang.Runtime");
          Constructor constructor = c1.getDeclaredConstructor();
          constructor.setAccessible(true);
          Object  object =  constructor.newInstance();
          Method method = c1.getMethod("exec", String.class);
           method.invoke(object,"/System/Applications/Calculator.app/Contents/MacOS/Calculator");
        } catch (Exception e){
           e.printStackTrace();
        }
    }
}


```

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210823001935.png)

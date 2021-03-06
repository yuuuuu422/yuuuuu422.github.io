---
title: Classcloader
key: Classcloader
---

## 类从编译到执行的过程

- 编译器将People.java源文件编译为People.class字节码文件
- ClassLoader 将字节码读入内存，转换为JVM中的Class<People>对象
- JVM利用Class<People>对象实例化为People对象

![image-20220116212751953](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/01/20220116212751.png)

以上图为例子，创建一个ClassLoaderTest.java文件运行，经过javac编译，然后生成ClassLoaderTest.class文件。这个java文件和生成的class文件都是存储在我们的磁盘当中。但如果我们需要将磁盘中的class文件在java虚拟机内存中运行，需要经过一系列的类的生命周期（加载、连接（验证-->准备-->解析）和初始化操作，最后就是我们的java虚拟机内存使用自身方法区中字节码二进制数据去引用堆区的Class对象。

通过这个流程图，我们就很清楚地了解到类的加载就是由java类加载器实现的，作用将类文件进行动态加载到java虚拟机内存中运行。

## 类加载器的分类

![img](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211121150537.png)

### 引导类加载器(BootstrapClassLoader)

引导类加载器(BootstrapClassLoader)，底层原生代码是C++语言编写，属于jvm一部分，不继承java.lang.ClassLoader类，也没有父加载器，主要负责加载核心java库(即JVM本身)，存储在/jre/lib/rt.jar目录当中。(同时处于安全考虑，BootstrapClassLoader只加载包名为java、javax、sun等开头的类)。

### 拓展类加载器(ExtensionsClassLoader)

扩展类加载器(ExtensionsClassLoader)，由`sun.misc.Launcher$ExtClassLoader`类实现，用来在/jre/lib/ext或者java.ext.dirs中指明的目录加载java的扩展库。Java虚拟机会提供一个扩展库目录，此加载器在目录里面查找并加载java类。

### App类加载器/系统类加载器（AppClassLoader/SystemClassloader）

App类加载器/系统类加载器（AppClassLoader），由`sun.misc.Launcher$AppClassLoader`实现，一般通过通过(java.class.path或者Classpath环境变量)来加载Java类，也就是我们常说的classpath路径。通常我们是使用这个加载类来加载Java应用类，可以使用`ClassLoader.getSystemClassLoader()`来获取它。

举一个例子来了解这三个个加载器的关系：

```java
package com.theoyu.classloader;

public class TestClassLoader{
    public static void main(String[] args) {
    ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
    System.out.println(systemClassLoader);//sun.misc.Launcher$AppClassLoader@18b4aac2

    ClassLoader extClassLoader = systemClassLoader.getParent();
    System.out.println(extClassLoader);//sun.misc.Launcher$ExtClassLoader@1b6d3586

    ClassLoader bootstrapClassLoader = extClassLoader.getParent();
    System.out.println(bootstrapClassLoader);//null

    ClassLoader classLoader=Class.class.getClassLoader();
    System.out.println(classLoader);//null
    }
}
```

Object类是所有子类的父类，归属于BootstrapClassLoader，拓展类加载器的上层引导同样也是BootstrapClassLoader，因为其不继承于ClassLoader，返回null也是理所当然。

## 双亲委派机制

其实上面的那一张图已经解释的很清楚了，ClassLoader **双亲委派机制**始终按照 ApplicationClassLoader -> ExtensionsClassLoader -> BootstrapClassLoader 顺序，BootStrap ClassLoader 优先级最高，以此类推。当 ClassLoader 接收到类加载请求时，首先将任务委托给其父类加载器来加载，如果父类加载器无法完成该请求，将由子类加载器来进行加载。双亲委派机制使得类有了层次划分，防止重复加载类以及保证核心类不被篡改。

## ClassLoader核心方法

### findLoadedClass

查找JVM已经加载过的类

```java
protected final Class<?> findLoadedClass(String name) {
    if (!checkName(name))
        return null;
    return findLoadedClass0(name);
}
```

### findClass

查找java的类，如果没有重写默认返回失败异常

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```

### defineClass

定义一个Java类，将字节码解析成虚拟机识别的Class对象

```java
protected final Class<?> defineClass(byte[] b, int off, int len)
    throws ClassFormatError
{
    return defineClass(null, b, off, len, null);
}
```

### resolveClass

链接指定Java类

```java
protected final void resolveClass(Class<?> c) {
        resolveClass0(c);
    }

    private native void resolveClass0(Class c);
```



### loadClass

加载指定的java类

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

可以看到，loadClass分为以下几个步骤：

1. `findLoadedClass()`检查类是否已被加载
2. 如果当前`classLoader` 传入了父类加载器则直接使用，否则使用JVM的`Bootstrap ClassLoader`加载
3. 如果上一步加载失败，调用自身的`findClass`方法尝试加载类
4. 如果当前的`ClassLoader`没有重写了`findClass`方法，那么直接返回类加载失败异常。如果当前类重写了`findClass`方法并找到了对应的类字节码，则调用`defineClass`方法去JVM中注册该类
5. 调用loadClass的时候传入的`resolve`参数为true，那么还需要调用`resolveClass`方法链接类,默认为false
6. 返回一个被JVM加载后的`java.lang.Class`类对象。

## 自定义类加载器

TestClass

```java
package com.theoyu.classloader;

public class TestClass {
    public String TestClass(){
        return "hello world~";
    }
}

```

ReadClassBytes

```java
package com.theoyu.classloader;
import java.io.*;
import java.util.Arrays;
public class ReadClassBytes {
    public static byte[]   Read(String filePath)  {
        InputStream input = null;
        BufferedInputStream bfs = null;
        byte[] buffer = new byte[1024];
        try {
            input = new FileInputStream(filePath);
            bfs = new BufferedInputStream(input);
            int bfsRead = bfs.read(buffer);
            return Arrays.copyOfRange(buffer,0,bfsRead);
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```
CustomClassLoader
```java
package com.theoyu.classloader;
import com.theoyu.classloader.ReadClassBytes;

import java.io.FileNotFoundException;

public class CustomClassLoader extends ClassLoader{
    private static String className="com.theoyu.classloader.TestClass";

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        if (name.equals(className)) {
            byte[] classBytes = ReadClassBytes.Read("src/com/theoyu/classloader/TestClass.class");
            return defineClass(className, classBytes, 0, classBytes.length);
        }
        return super.findClass(name);
    }

    public static void main(String[] args) {
        CustomClassLoader customClassLoader=new CustomClassLoader();
        try {
            Class<?> clazz = Class.forName(className,true,customClassLoader);
            Object obj = clazz.newInstance();
            System.out.println(obj.getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

后面获取对象方法涉及到反射的知识，只需获取到返回类对象即可。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210818000545.png)

## UrlClassLoader

`URLClassLoader`继承了`ClassLoader`，`URLClassLoader`提供了加载远程资源的能力，在写漏洞利用的`payload`或者`webshell`的时候我们可以使用这个特性来加载远程的jar来实现远程的类方法调用。

CMD.java 并部署CMD.jar在tomcat根目录下

```java	
package com.theoyu.classloader;

import java.io.IOException;

public class CMD {
    public static Process exec(String cmd)throws IOException{
        return Runtime.getRuntime().exec(cmd);
    }
}
```

TestUrlClassLoader.java

```java
package com.theoyu.classloader;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.net.URL;
import java.net.URLClassLoader;

public class TestUrlClassLoader {
    public static void main(String[] args) {
        try {
            URL url = new URL("http://127.0.0.1:8080/CMD.jar");
            URLClassLoader ucl = new URLClassLoader(new URL[]{url});
            String cmd = "ls";
            Class cmdClass = ucl.loadClass("com.theoyu.classloader.CMD");
            Process process = (Process) cmdClass.getMethod("exec", String.class).invoke(null, cmd);
            InputStream           in   = process.getInputStream();
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[]                b    = new byte[1024];
            int                   a    = -1;

            // read command execute result
            while ((a = ((InputStream) in).read(b)) != -1) {
                baos.write(b, 0, a);
            }
            System.out.println(baos.toString());
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```






























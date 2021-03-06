---
title: Rmi
key: Rmi
---

## 概述

**RMI ( Remote Method Invocation , 远程方法调用 )**能够让在某个 Java虚拟机 上的对象像调用本地对象一样调用另一个 Java虚拟机对象上的方法 , 这两个 Java虚拟机 可以是运行在同一台计算机上的不同进程, 也可以是运行在网络中不同的计算机上 .

说到底， RMI是专为Java环境设计的远程方法调用机制，是一种用于实现远程调用（RPC，Remote Procedure Call）的一组Java API。之前有学习过[gRPC的调用方法](https://theoyu.top/posts/course/rpc/),也算是大同小异。

## RMI简单实现

 RMI的设计模式中，主要包括以下三个部分的角色：

- Server：服务端，远程方法的提供者，并向Registry注册自身提供的服务。

- Registry：提供服务注册与服务获取。即Server端向Registry注册服务，比如地址、端口等一些信息，Client端从Registry获取远程对象的一些信息，如地址、端口等，然后进行远程调用。
- Client：客户端，远程方法的消费者，从Registry获取远程方法的相关信息并且调用。

具体案例:

```
🌀  tree  
├── client
│   └── HelloClient.java
└── server
    ├── Hello.java
    ├── HelloImpl.java
    └── HelloServer.java

```

### 定义远程接口

```java
package com.theoyu.rmi.server;

import java.rmi.Remote;
import java.rmi.RemoteException;

public  interface Hello extends Remote {
    String sayHello(String name) throws RemoteException;
}
```

只有继承了**Remote**的接口中的方法，才可以被远程调用。

由于远程调用的本质依旧是 " 网络通信 " . 而网络通信是经常出现异常的 . 因此 , 继承 **Remote** 接口的接口的所有方法必须要抛出 `RemoteException` 异常 . 事实上 , `RemoteException` 也是继承于 `IOException` 。

### 接口实现

```java
package com.theoyu.rmi.server;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class HelloImpl extends UnicastRemoteObject implements Hello {
    protected HelloImpl() throws RemoteException {
    }

    @Override
    public String sayHello(String name) throws RemoteException {
        return "Hello "+name;
    }
}
```

以下转自**Epicccal**

>实现类必须要继承 **`UnicastRemoteObject`** 类 
>
>**只有当接口的实现类继承了 `UnicastRemoteObject` 类 , 客户端访问获得远程对象时 , 远程对象才将会把自身的一个拷贝以 `Socket` 的形式传输给客户端**，这个拷贝也就是 `Stub` , 或者叫做 " 存根 " .
>
>准确的说 , **`java.rmi.server.UnicastRemoteObject` 类的构造函数将生成 `Stub` 和 `Skeleton`** . 而继承该类的子类将会在实例化时自动执行父类的构造函数 , 从而也生成 `Stub` 和 `Skeleton` .
>
>**这个 `Stub` 可以看作是远程对象在本地的一个代理 , 其中包含了远程对象的具体信息 . 客户端可以通过这个代理与服务端进行交互 .**
>
>**`Skeleton` 也叫做 " 骨架 " , 可以看作是服务端的一个代理 , 用来处理 `Stub` 发来的请求 , 然后去调用客户端真正需求的方法 , 然后再将方法执行结果返回给 `Stub`** .
>
>其实 , 与其说是客户端和服务端进行交互 , 不如说是 **客户端代理( `Stub` ) 和 服务端代理( `Skeleton` ) 在进行交互 .**

### 服务端配置

```java
package com.theoyu.rmi.server;

import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;


public class HelloServer {
    public static void main(String[] args) {
        try {
          //创建远程对象HelloImpl的实例对象
            Hello h =new HelloImpl();
          //创建Registry
            LocateRegistry.createRegistry(1099);
            Registry registry=LocateRegistry.getRegistry();
          //绑定HelloImpl对象至Registry
            registry.bind("hello",h);
            System.out.println("[+] RmiServer start");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

这里客户端可以通过`rmi://localhost:1099/hello`直接访问远程对象，不需要知道远程实例对象的名称。RMIServer将对象绑定在了Registry上,并且公开了一个固定的路径 ,供客户端访问。

### 客户端

```java
package com.theoyu.rmi.client;

import com.theoyu.rmi.server.Hello;

import java.rmi.Naming;

public class HelloClient {
    public static void main(String[] args) {
        try {
            Hello h = (Hello) Naming.lookup("rmi://localhost:1099/hello");
            System.out.println(h.sayHello("Theoyu"));
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

客户端只需要调用 `java.rmi.Naming.lookup` 函数 , 通过公开的路径从 Registry 上拿到对应接口的实现类 , 拿到实现类后 , 在通过本地接口即可调用远程对象的方法 .

因此 , 只需要一个接口 , 一个客户端连接程序 , 即可实现 JAVA 远程调用 .

## 后续

因为在整个RMI机制过程中，都是进行反序列化传输，我们可以利用这个特性使用RMI机制来对RMI远程服务器进行反序列化攻击。
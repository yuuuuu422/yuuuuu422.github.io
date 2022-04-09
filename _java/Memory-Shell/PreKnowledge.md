---
titile: 前言
---

## Tomcat 的 4 类容器组件：

Engine、Host 、Context 、 Wrapper：

- **Engine**（org.apache.catalina.core.StandardEngine）：最大的容器组件，可以容纳多个 Host。
- **Host**（org.apache.catalina.core.StandardHost）：一个 Host 代表一个虚拟主机，一个Host可以包含多个 Context。
- **Context**（org.apache.catalina.core.StandardContext）：一个 Context 代表一个 Web 应用，其下可以包含多个 Wrapper。
- **Wrapper**（org.apache.catalina.core.StandardWrapper）：一个 Wrapper 代表一个 Servlet（**重点** ：上文提到的动态注册Servlet组件的内存马技术，想要动态的去注册Servlet组件实现过程中的关键之一就是如何获取Wrapper对像，再往上也就是如何获取到Context对象，从而掌握整个Web应用）。

## Servlet的三大基础组件：

Servlet、Filter 、Listener ：

请求 → Listener → Filter → Servlet

- **Servlet**: 最基础的控制层组件，用于动态处理前端传递过来的请求，每一个Servlet都可以理解成运行在服务器上的一个java程序；生命周期：从Tomcat的Web容器启动开始，到服务器停止调用其destroy()结束；驻留在内存里面
- **Filter**：过滤器，过滤一些非法请求或不当请求，一个Web应用中一般是一个filterChain链式调用其doFilter()方法，存在一个顺序问题。
- **Listener**：监听器，以ServletRequestListener为例，ServletRequestListener主要用于监听ServletRequest对象的创建和销毁,一个ServletRequest可以注册多个ServletRequestListener接口（都有request来都会触发这个）。

## Context对象获取

>对于Tomcat来说，一个Web应用中的Context组件为org.apache.catalina.core.StandardContext对象，前文也有提到我们在实现**通过动态注册Servlet组件的内存马技术**的时候，其中一个关键点就是怎么获取器Context对象，而该对象就是StandContext对象。那么我们可以通过哪些途径得到StandContext对象那呢？

### 有requet对象的时候

> Tomcat中Web应用中获取的request.getServletContext是ApplicationContextFacade对象。该对象对ApplicationContext进行了封装，而ApplicationContext实例中又包含了StandardContext实例，所以当request存在的时候我们可以通过反射来获取StandardContext对象

#### 没有request对象的时候

1、不存在request的时候从currentThread中的ContextClassLoader中获取（适用Tomcat 8，9）

2、ThreadLocal中获取

3、从MBean中获取



## 内存马的分类

更具内存马的实现技术来分类将内存马分为三类：

1、基于动态添加Servlet组件的内存马

servlet

>- 1、首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象），上文中有提到获取StandardContext获取的方式，这里因为是jsp来实现的，所以可以直接拿到request，所以就利用上文提到的第一种方法，通过反射即可获取到Tomcat下的StrandardContext对象，下面也是同理。
>- 2、注册一个Servlet对象并重写其Service方法，在其中实现命令执行并通过response返回
>- 3、创建Wrapper对象并利用各个船舰的Servlet来初始化
>- 4、为Wrapper添加路由映射

filter

>- 1、首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）
>- 2、利用获取的上下文StandardContext对象获取filterconfigs对象
>- 3、创建一个恶意Filter对象并重写其doFilter方法，在其中实现命令执行并通过response返回，最后filterChain传入后面的filter
>- 4、创建FilterDef对象并利用刚刚创建的Filter对象来初始化，并新建一个FilterMap对象，为创建的FilterDef对象添加URL映射
>- 5、利用创建FilterConfig对象并使用刚刚创建的FilterDef对象初始化，最后将其加入FilterConfigs里面，等待filterChain.dofilter调用

##### Listener

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）
2. 创建一个ServletRequestListener对象并重写其requestDestroyed方法，在其中实现命令执行并通过response回显.
3. 将创建的ServletRequestListener对象通过StandardContext添加到事件监听中去



2、基于动态添加框架组件的内存马

这里说到的框架有很多，spring、springboot、weblogic等

3、基于Javaagent和Javassist技术的内存马
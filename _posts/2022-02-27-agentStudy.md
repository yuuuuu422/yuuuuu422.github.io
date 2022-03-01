---
title: 记一次Agent之旅
---

**Agent**是字节码插桩不可忽视的机制，无论是在软件破解，还是现在比较流行的动态交互安全监控领域都少不了它的身影。

<!--more-->

例子来源于知识盒子：攻击[Java Web应用 - Agent 实现破解License示例](https://zhishihezi.net/endpoint/richtext/8b199096a7603e780cd9f25343b7e916?event=436b34f44b9f95fd3aa8667f1ad451b173526ab5441d9f64bd62d183bed109b0ea1aaaa23c5207a446fa6de9f588db3958e8cd5c825d7d5216199d64338d9d00fdf64cc2c4da297406094f5b661801902ac51f8888b3391a256f270e8a68376eeca1c3c5ccd0e597d1d42b9501c3ff283fead46350a5a31976ac9e27c13a199bfac416522d27f5a3102df757354753a75bae7449a62604a33334660cc0d13020b8b6fb47151aec6d7373e22f2f6f79606a767bfd979bbaa23839d00ced3e57bd6d960016e14a398720e5bfcceaec529d59e7fee481ee0f82cd178319acf489f5ba542390e551edfd4bed88ba1fea85e544573ac41b0e0bdd0beff7fc52d9b113#3)，在这个示例中，作者模拟了一个 **License** 校验场景，每一秒就会调用 `checkExpiry`函数检测授权是否过期：

```java
public class CheckLicense {
    private static final SimpleDateFormat DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    private static boolean checkExpiry(String expireDate) throws ParseException {
        try {
            Date date = DATE_FORMAT.parse(expireDate);
            // 检测当前系统时间早于License授权截至时间
            if (new Date().before(date)) {
                return false;
            }
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return true;
    }

    public static void main(String[] args) {
        // 设置一个已经过期的License时间
        final String expireDate = "2020-10-01 00:00:00";

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        String time = "[" + DATE_FORMAT.format(new Date()) + "] ";
                        // 检测license是否已经过期
                        try {
                            if (checkExpiry(expireDate)) {
                                System.err.println(time + "您的授权已过期，请重新购买授权！");
                            } else {
                                System.out.println(time + "您的授权正常，截止时间为：" + expireDate);
                            }
                        } catch (ParseException e) {
                            e.printStackTrace();
                        }

                        // sleep 1秒
                        TimeUnit.SECONDS.sleep(1);

                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

![image-20220227231624883](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102845.png)

我们有两种方式启动Agent进行插桩：

- 实现`premain`方法，在JVM启动前加载。
- 实现`agentmain`方法，在JVM启动后加载。

## 启动前指定Agent位置

在目标JVM启动的同时加载Agent，需要实现`premain`，再通过自定义**ClassFileTransformer**去hook相应的方法。

结合`checkExpiry`代码，我们想要绕过检测，应该是函数检测是否过期永远返回false，那么我们用**javassist**有两种方式修改：

-  `ctMethod.insertBefore("return false;");`

- `ctMethod.insertAfter("return false;");`

前者相当于直接进入函数就执行return，第二种是在所有return语句前加入，这里我们选用前者。

代码：

```java
public class MyAgent3 {
    private static final String HookClass = "CheckLicense";
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new MyClassFileTransformer());
    }

    public static class MyClassFileTransformer implements ClassFileTransformer{

        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) {
            String newClassName = className.replace("/",".");
            if(newClassName.equals(HookClass)){
                System.out.println("Hook: "+HookClass);
                ClassPool classPool = ClassPool.getDefault();
                try {
                    CtClass ctClass = classPool.makeClass(new ByteArrayInputStream(classfileBuffer));
                    CtMethod ctMethod = ctClass.getDeclaredMethod("checkExpiry",new CtClass[]{classPool.getCtClass("java.lang.String")});
                    ctMethod.insertBefore("return false;");
                    classfileBuffer = ctClass.toBytecode();

                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            return classfileBuffer;
        }
    }
}
```

pom.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.example</groupId>
    <artifactId>agent</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.12.1.GA</version>
        </dependency>
    </dependencies>
<build>
    <!-- 最终编译JAR名字 -->
    <finalName>MyAgent</finalName>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>2.3.2</version>
            <configuration>
                <archive>
                    <manifestEntries>
                        <Premain-Class>top.theoyu.MyAgent3</Premain-Class>
                    </manifestEntries>
                </archive>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <artifactSet>
                    <includes>
                        <include>javassist:javassist:jar:</include>
                    </includes>
                </artifactSet>
            </configuration>
        </plugin>
    </plugins>
</build>
</project>
```

这里用了**maven-jar-plugin**和**maven-shade-plugin**两个插件，前者需指定**MANIFEST.MF**文件路径，或直接用`<manifestEntries>`设置。后者用于复制原依赖到jar包。

`mvn clearn install`打包，然后执行 `java -javaagent:MyAgent.jar CheckLicense`

![image-20220228000354361](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102856.png)

报错了，我们把修改一下源码，加上`ctClass.writeFile();`，把hook后的字节码输出看一下：

从反编译的源码上来看，的确是已经达到了`ctMethod.insertBefore("return false;");`的目的。

![image-20220228000814337](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102902.png)

从字节码的角度，也没有问题：

![image-20220228001152235](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102907.png)

那么问题出在哪呢？通过`Expecting a stack map frame`关键字，我在[geeksforgeeks](https://www.geeksforgeeks.org/verification-java-jvm/)上找到了答案：

>After the class loader in the [JVM ](https://www.geeksforgeeks.org/jvm-works-jvm-architecture/)loads the byte code of .class file to the machine the Bytecode is first checked for validity by the verifier and this process is called as **verification**. The verifier performs as much checking as possible at the Linking so that expensive operation performed by the interpreter at the run time can be eliminated. It enhances the performances of the interpreter.

**Some of the checks that verifier performs:**

- Uninitialized Variables
- Access rules for private data and methods are not violated.
- Method calls match the object Reference.
- There are no operand stack overflows or underflows.
- The arguments to all the Java Virtual Machine instructions are of valid types.
- Ensuring that final classes are not subclassed and that final methods are not overridden
- Checking that all field references and method references have valid names, valid classes, and a valid type descriptor. ([source](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.10))

顺着类加载机制，在《深入理解Java虚拟机》7.3.2同样也有所解释：

![image-20220228010052451](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102915.png)

回到`StackMapTable`,在跳转指令中我们只有26和34两个locals，但是这里的却多了一个29的Offset，在jvm的**verification**中将会报错。

![F546A348-F473-4899-9917-A7A6A72510A3](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102910.png)

解决的方法，就是跳过类加载的验证过程，` -noverify`:

![image-20220228010730883](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228102920.png)

但是这样解决总还是有些说不过去， 实际上我们可以直接使用	`ctMethod.setBody("return false;");`去把整个方法抽空，再插入代码，这样就避免了`StackMapTable`所导致的验证失败。

或者使用颗粒度更细化的**ASM**框架处理，直接定位到最后一个`ireturn`指令前的`iconst_1`，修改为`iconst_0` 即可。

## 启动后进行Agent Attach

JDK 1.6 新增了attach (附加方式)方式，可以对运行中的 Java 进程附加 Agent 。

 上代码：

```java
public class MyAgent4 {
    public static  String HookClass = "CheckLicense" ;
    public static void main(String[] args)throws Exception {
        // 拿到agent的绝对路径
        URL agentURL = MyAgent4.class.getProtectionDomain().getCodeSource().getLocation();
        String agentPath = new File(agentURL.toURI()).getAbsolutePath();

        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor desc : list) {
            if(desc.displayName().equals(HookClass)){
                String pid = desc.id();
                System.out.println("进程名称：" + HookClass+", 进程ID：" + pid);
                VirtualMachine vm = VirtualMachine.attach(pid);
                vm.loadAgent(agentPath);
                vm.detach();
            }
        }
    }
    public static void agentmain(String args, final Instrumentation inst) {
        // 添加自定义的Transformer，第二个参数true表示是否允许Agent Retransform，
        // 需配合MANIFEST.MF中的Can-Retransform-Classes: true配置
        inst.addTransformer(new MyClassFileTransformer(), true);
        // 获取所有已经被JVM加载的类对象
        Class[] loadedClass = inst.getAllLoadedClasses();
        for (Class clazz : loadedClass) {
            String className = clazz.getName();
            if (inst.isModifiableClass(clazz)) {
                // 使用Agent重新加载HelloWorld类的字节码
                if (className.equals(HookClass)) {
                    try {
                        inst.retransformClasses(clazz);
                    }catch (UnmodifiableClassException e){
                        e.printStackTrace();
                    }
                }
            }
        }
    }
    public static class MyClassFileTransformer implements ClassFileTransformer {
        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) {
            String newClassName = className.replace("/",".");
            if(newClassName.equals(HookClass)){
                System.out.println("Hook: "+HookClass);
                ClassPool classPool = ClassPool.getDefault();
                try {
                    CtClass ctClass = classPool.makeClass(new ByteArrayInputStream(classfileBuffer));
                    CtMethod ctMethod = ctClass.getDeclaredMethod("checkExpiry",new CtClass[]{classPool.getCtClass("java.lang.String")});
                    ctMethod.setBody("return false;");
                    classfileBuffer = ctClass.toBytecode();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            return classfileBuffer;
        }
    }
}
```

![image-20220228152258772](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228152303.png)

结果：

![image-20220228152539133](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220228152543.png)

`attach`需要用到tools.jar，但是shade插件打包的时候好像不能copy系统自带的包，所以最后还是用`-Xbootclasspath/a:$JAVA_HOME/lib/tools.jar`去指定classpath了。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.example</groupId>
    <artifactId>agent</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.sun</groupId>
            <artifactId>tools</artifactId>
            <version>${java.version}</version>
            <scope>system</scope>
            <systemPath>${java.home}/../lib/tools.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.12.1.GA</version>
        </dependency>
    </dependencies>
<build>
    <!-- 最终编译JAR名字 -->
    <finalName>MyAgent</finalName>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <manifestEntries>
                                    <addClasspath>true</addClasspath>
                                    <Main-Class>top.theoyu.MyAgent4</Main-Class>
                                    <Agent-Class>top.theoyu.MyAgent4</Agent-Class>
                                    <Can-Redefine-Classes>true</Can-Redefine-Classes>
                                    <Can-Retransform-Classes>true</Can-Retransform-Classes>
                                </manifestEntries>
                            </transformer>
                        </transformers>
                        <artifactSet>
                            <includes>
                                <include>javassist:javassist:jar:</include>
                            </includes>
                        </artifactSet>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
</project>
```


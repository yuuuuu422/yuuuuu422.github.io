---
title: 浅谈jvm和字节码
---

第一部分，我们简单谈一谈jvm解释执行的依据；第二部分关于字节码，也算是对《深入理解Java虚拟机》一书的实践；最后会简单介绍两种字节码操作框架，以实现字节码插桩。

<!--more-->

## jvm解释执行

![image-20220209190348623](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220209190348.png)     

在Java语言中，Javac编译器完成了程序代码经过词法分析、语法分析到抽象语法树，再遍历语法树生成线性的字节码指令流的过程。

执行引擎在解析字节码文件的时候，并不只有解释执行一种情况，也可以通过编译执行( 即通过即时编译器产生本地代码执行 )，这种方式也称为 [jit](https://aboullaite.me/understanding-jit-compiler-just-in-time-compiler/) 。不过无论哪种方式，输入的都是字节码二进制流，输出的都是执行结果，所以处理的过程其实都是等效的，区别只在于执行速度有所不同。

### 基于栈的字节码解释执行引擎

关于字节码的解释执行，其实下面的伪代码就可以说明：

```c++
do {
    获取下一个指令
    解释指令
} while (还有指令);
```

谈到指令，就离不开指令集的架构，笔者二进制方面较弱，所以也只用通俗易懂(~~抽象~~)的语言来描述。

我们分别用**基于寄存器的方案**和**基于栈的方案**，来表示`1+1`：

**基于寄存器的方案**：

```
mov eax,1
add eax,1
```

**基于栈的方案**：

```
push_1
push_1
add 
```

如同《 深入理解Java虚拟机 》所说的：『 Java输出的字节码指令流，**基本上**是一种基于栈的指令集架构 』。为什么有**基本上**三个字呢？因为纯粹基于栈的指令集架构应当全部是零地址指令，也就是不存在显式参数的。如果你对PVM(Python Virtual Machine)有所了解，知道Python解释执行分为栈区(Stack)和存储区(Memo)两大块，JVM也不列外，其使用**局部变量表**辅助栈区执行。

**局部变量表**：栈帧内部的数据结构, 是个数组. 通过数组位置访问，换个说法也可以当作可以特殊的**寄存器**。

那么JVM解释执行的方式，其实可以当作**栈和寄存器混合执行**来看待。

我们把下列代码转化为JVM指令看看

```c
int a = 1 + 1;
int b = 2 + 2;
int c = 3;
int d = b - a;
d = d - c;
System.out.println(d);
```

其中类似`1+1`的指令，在前端就已经被javac优化，下列指令中，我们用分别把栈帧和局部变量表表示一下(前面为栈帧，左边为栈顶；后面为局部变量表)

```c
 0 iconst_2 // (2)
 1 istore_1 // ()      {1:2}
 2 iconst_4 // (4)     {1:2}
 3 istore_2 // ()      {1:2, 2:4}
 4 iconst_3 // (3)
 5 istore_3 // ()      {1:2, 2:4, 3:3}
 6 iload_2  // (4)     {1:2, 2:4, 3:3}
 7 iload_1  // (2,4)   {1:2, 2:4, 3:3}
 8 isub     // (2)     {1:2, 2:4, 3:3}
 9 istore 4 // ()      {1:2, 2:4, 3:3, 4:2}
11 iload 4  // (2)     {1:2, 2:4, 3:3, 4:2}
13 iload_3  // (3,2)   {1:2, 2:4, 3:3, 4:2}
14 isub     // (-1)    {1:2, 2:4, 3:3, 4:2}
15 istore 4 // ()      {1:2, 2:4, 3:3, 4:-1}
17 getstatic #2 <java/lang/System.out : Ljava/io/PrintStream;>  // (java/lang/System.out : Ljava/io/PrintStream;) {1:2, 2:4, 3:3, 4:-1}
20 iload 4  // (-1,java/lang/System.out : Ljava/io/PrintStream;)
22 invokevirtual #3 <java/io/PrintStream.println : (I)V> // println(-1)
25 return
```

纯粹基于栈的方案，貌似没有，因为只有`pop`,`push`操作的话，在局部变量较多的情况下，需要频繁的搬运数据，防止之前的局部变量消失。

我们所说的指令，也就是字节码指令，其从Class文件中解析出来，Class文件本身是静态的，解析Class文件不是什么高深的技术，在 [Java虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se8/html/) 下我们可以很好的理解， 下面我们通过几个字节码查看工具走近字节码。

## 走近字节码

Class文件是一组以字节为基础单位的二进制流，各个数据严格按照顺序紧凑排列在文件中，我们可以依靠一些工具反编译二进制流，得到详细的数据信息和指令。

反汇编(disassembly)和反编译(decompile)是两个不等同的概念，在java中，反汇编是指是将.class文件转换成opcode，反编译是指将.class文件转换为.java文件，是更加高级的体现。IDEA自带的反编译就非常强大，基本上可以完全复原代码，不过本文是站在字节码角度的分析，所以还是更加侧重于前者。

**javap**

javap是jdk自带的反解析工具。它的作用就是根据class字节码文件，反解析出当前类对应的code区（汇编指令）、本地变量表、异常表和代码行偏移量映射表、常量池等等信息。

```txt
javap -help                          
用法: javap <options> <classes>
其中, 可能的选项包括:
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```

后续我们对以下代码的Class文件进行反编译

```java
package demo;

public class Test2 {
    public static void main(String[] args) {
        int a = 10;
        String name = "theoyu";
        say(name);
    }
    static void say(String name){
        System.out.println(name);
    }
}
```

```java
🌀  classes  javap  -c demo.Test2
Compiled from "Test2.java"
public class demo.Test2 {
  public demo.Test2();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10
       2: istore_1
       3: ldc           #2                  // String theoyu
       5: astore_2
       6: aload_2
       7: invokestatic  #3                  // Method say:(Ljava/lang/String;)V
      10: return

  public static void say(java.lang.String);
    Code:
       0: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: aload_0
       4: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       7: return
}
```

如果需要查看常量池，局部变量表等信息，可以用`javap - v`打印

```
javap  -v demo.Test2
Constant pool:
   #1 = Methodref          #7.#27         // java/lang/Object."<init>":()V
   #2 = String             #28            // theoyu
   #3 = Methodref          #6.#29         // demo/Test2.say:(Ljava/lang/String;)V
...
...
 LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  args   [Ljava/lang/String;
            3       8     1     a   I
            6       5     2  name   Ljava/lang/String;
...
...
```

当然这些概念我们会在后续介绍。idea支持的jclasslib工具可以可视化查看字节码文件，并且支持直接跳转到《Java虚拟机规范》中查看陌生指令，以及直接修改操作码。

**jclasslib**

在idea插件中下载jclasslib后，就可以直接打开class文件查看

![image-20220210155153889](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220210155153.png)

**classpy && ClassViewer**

[classpy ](https://github.com/zxh0/classpy)是《自己动手写Java虚拟机》一书作者写的查看class文件的gui工具，后续还拓展了lua、wasm等文件格式，不过其兼容性不是很好，可以用更加精简美观的[ClassViewer](https://github.com/ClassViewer/ClassViewer)代替。

**ClassViewer**：

![image-20220210161545940](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220210161545.png)

字节码文件按照以下10个部分的固定顺序组成。

![image-20220210170340077](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220210170340.png)

根据《Java虚拟机规范》的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型:“无符号数”和“表”，无符号数属于基本数据结构，表由多个无符号数或者其他表作为数据项构成，为了区分，表的命名以**_info**结尾，所以上述10个部分又可以列作为：

```c
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

接下来我们一一介绍这10个部分：

### (1) 魔数(Magic Number)

魔数也就是一个标识头，占用四个字节，其值为**『cafebaby』**，说明这是一个字节码文件。

常见的文件头还有zip文件头**504B0304**，JPEG文件头**FFD8FF**等。

![image-20220210180039782](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220210180039.png)

### (2)版本号(Version)

版本号为魔数之后的4个字节，前两个字节表示次版本号（Minor Version），后两个字节表示主版本号（Major Version），如上图中的**『00 00 00 34』** ,主版本号化为10进制为52，代表JDK1.8，以此类推51即代表JDK1.7。

### (3)常量池(Constant Pool)

常量池是是Class文件里的资源仓库，首先是常量池容量计数器(constant_pool_count)，说白了就是用来记录常量池中常量的个数，用2个字节来记录。

![image-20220212104824796](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220212104824.png)

这里常量池容量计数器值为为41，但我们打印发现实际上常量个数只有40个：

```
Constant pool:
   #1 = Methodref          #7.#27         // java/lang/Object."<init>":()V
   #2 = String             #28            // theoyu
   #3 = Methodref          #6.#29         // demo/Test2.say:(Ljava/lang/String;)V
   #4 = Fieldref           #30.#31        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = Methodref          #32.#33        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = Class              #34            // demo/Test2
   #7 = Class              #35            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               LocalVariableTable
  #13 = Utf8               this
  #14 = Utf8               Ldemo/Test2;
  #15 = Utf8               main
  #16 = Utf8               ([Ljava/lang/String;)V
  #17 = Utf8               args
  #18 = Utf8               [Ljava/lang/String;
  #19 = Utf8               a
  #20 = Utf8               I
  #21 = Utf8               name
  #22 = Utf8               Ljava/lang/String;
  #23 = Utf8               say
  #24 = Utf8               (Ljava/lang/String;)V
  #25 = Utf8               SourceFile
  #26 = Utf8               Test2.java
  #27 = NameAndType        #8:#9          // "<init>":()V
  #28 = Utf8               theoyu
  #29 = NameAndType        #23:#24        // say:(Ljava/lang/String;)V
  #30 = Class              #36            // java/lang/System
  #31 = NameAndType        #37:#38        // out:Ljava/io/PrintStream;
  #32 = Class              #39            // java/io/PrintStream
  #33 = NameAndType        #40:#24        // println:(Ljava/lang/String;)V
  #34 = Utf8               demo/Test2
  #35 = Utf8               java/lang/Object
  #36 = Utf8               java/lang/System
  #37 = Utf8               out
  #38 = Utf8               Ljava/io/PrintStream;
  #39 = Utf8               java/io/PrintStream
  #40 = Utf8               println
```

这是因为常量池中的常量计数是从1开始的，默认第0个是null。

所以数据区是由 **constant_pool_count-1** 个cp_info表组成，在 jdk1.8版本的字节码中共有14种类型的cp_info( jdk16更新为17种 )，每种类型的结构都是固定的。

![图6 各类型的cp_info](https://p0.meituan.net/travelcube/f5bdc7e8203ec666a531fcd19cdbcddc519208.png)

我们以最为**CONSTANT_utf8_info**为例，其tag为01，对应utf8类型，接下来两个字节标识该字符串的长度Length，然后Length个字节为这个字符串具体的值。

![image-20220212115637072](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220212115637.png)

其他类型的cp_info不再赘述，整体结构大同小异，总的来说就是以下几个过程：

1. 第一步：先找tag位

2. 第二步：根据tag的值从常量项表中找到对应的常量项结构

3. 第三步：根据常量项的结构，我们找出对应的字节

4. 第四步：根据字节，转化为具体值

### (4)访问标志(access_flag)

![image-20220212124300303](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220212124300.png)

常量池结束之后的两个字节，描述该Class是类还是接口，以及是否被Public、Abstract、Final等修饰符修饰。

![image-20220212124411633](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220212124411.png)

如上图 **00 21**，其还可以表示一种组合，是**0x0001+0x0020** ，即**ACC_PUBLIC和ACC_SUPER**。

### (5 6 7)类索引、父类索引与接口索引集合

在类的访问标志下方就是**类索引**，占2个字节，在字节码中找到是**00 06**，它的涵义是索引，所以我们就去常量池表中找索引为6的值，发现指向的就是**demo/Test2**的索引 

![image-20220212130645934](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220212130645.png)

父类索引同理，父类索引后为两字节的接口计数器，描述了该类或父类实现的接口数量。紧接着的n个字节是所有接口名称的字符串常量的索引值。

### (8)字段表(fileds)

字段表用于描述类和接口中声明的变量，包含类级别的变量以及实例变量，但是不包含方法内部声明的局部变量。同常量池，字段表也分为两部分，第一部分两个字节为fields_count，描述字段个数；第二部分是fields_count个字段的详细信息fields_info。

上述的代码因为变量都是写在函数内，为局部变量，不存在字段，我们以下述代码为例：

```java
package demo;

public  class Test3 {
    private int age ;
}
```

![image-20220213161459739](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220213161459.png)

最开始的两个字节为**访问标志**，这个我们已经比较熟悉了，往后分别是**字段名称**和**字段描述符**，对应的都是常量池的索引，可以查询其值。

再往后的两个字节是属性表个数，如果一个字段被 **final static** 、**volatile** 等关键字修饰，比如 ` final static public int age = 100`，那么属性表中还会有一项称为 **ConstantValue** 的属性，其值指向常量 100 ，关于属性表后续还会介绍。

### (9)方法表(metheds)

字段表结束后为方法表，方法表也是由两部分组成，前两个字节描述方法的个数；第二部分为每个方法的详细信息。方法的详细信息较为复杂，包括方法的访问标志、方法名、方法的描述符以及方法的属性，如下图所示：

![图12 方法表结构](https://p0.meituan.net/travelcube/d84d5397da84005d9e21d5289afa29e755614.png)

再拿这一段代码看看，在这个案例中一共有 **构造方法、main方法、say** 三个方法

```java
package demo;

public class Test2 {
    public static void main(String[] args) {
        int a = 10;
        String name = "theoyu";
        say(name);
    }
    static void say(String name){
        System.out.println(name);
    }
}
```

![image-20220213164330423](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220213164330.png)

方法的权限修饰符和之前的访问权限大同小异，方法名和方法的描述符都是常量池中的索引值，可以通过索引值在常量池中找到。所以我们把重点放在**方法的Code属性表**这一部分。

![image-20220213164711916](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220213164711.png)

![image-20220213173739500](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220213173739.png)

**attribute_name_index**是一项指向**CONSTANT_Utf8_info**型常量的索引，此常量值固定为“Code”，**attribute_length** 代表属性值的长度。

**max_stack**代表操作数栈(**Operand Stack**)深度的最大值，虚拟机运行的时候需要根据这个值来分配栈帧(Stack Frame)中的操作栈深度。

**max_locals**代表了局部变量表所需的存储空间。在这里，**max_locals**的单位是变量槽(Slot)，变量槽是虚拟机为局部变量分配内存所使用的最小单位。在main函数中，一共有**args、a、name** 三个局部变量，所以大小为3。

**code_length** 和 **code** 用于存储Java源程序编译后生成的字节码指令。

**attributes**属性表中存储有**LineNumberTable**和**LocalVariableTable**两个重要属性：

- “LineNumberTable”：行号表，将Code区的操作码和源代码中的行号对应，Debug时会起到作用（源代码走一行，需要走多少个JVM指令操作码）。
- “LocalVariableTable”：局部变量表（也叫本地变量表），包含This和局部变量，之所以可以在每一个方法内部都可以调用This，是因为JVM将This作为每一个方法的第一个参数隐式进行传入。

我们结合指令和局部变量表对下述代码的main函数分析，先提一下局部变量表的start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖 的长度，两者结合起来就是这个局部变量在字节码之中的作用域范围。

![image-20220213205101911](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220213205102.png)



- 0 `bipush 10`：把 10 放在栈顶
- 2 `istore_1`  ：把栈顶元素 10 存储到局部变量表序号 1，name_index 对应常量池索引19，其值为 a ，类型为 int ，对应代码 `int a = 10`，此指令结束代表局部变量 a 的生命周期开始，也就是对应 **pc 3**。
- 3 `ldc #2` ：    把常量池索引为2的值( 字符串『theoyu』 ) 放在栈顶
- 5 `astore_2` ： 把栈顶元素存储到局部变量表序号 2 ，name_index 对应常量池索引21，其值为 name ，类型为 String ，对应代码 `String name = "theoyu"`，此指令结束代表局部变量 name 的生命周期开始，也就是对应 **pc 6**。
- 6 `aload_2`：引用 局部变量表 2的值『theoyu』到栈顶。
-  7 `invokestatic #3` 调用 static method `demo/Test2.say`，传入栈顶参数『theoyu』
- 10 `return`

 所以整个字节码解释执行可以用以下伪代码描述：

```c
do {
  自动计算PC寄存器的值加1; 
  根据PC寄存器指示的位置，从字节码流中取出操作码; 
  if (字节码存在操作数) 从字节码流中取出操作数; 
  执行操作码所定义的操作;
} while (字节码流长度 > 0);
```

### (10)附加属性表(attributes)

字节码的最后一部分，该项存放了在该文件中类或接口所定义属性的基本信息。

## 字节码修改

在我们了解了字节码结构后，只用简单的文本编辑器，甚至你只需要一个vim，就可以随意修改字节码文件，但是这莫过于有些麻烦，而一些较为上层的框架就为我们提供了修改已有字节码、动态生成全新字节码的功能。

### ASM

ASM 库提供了两个用于生成和转换已编译类的 API，一个是Core API，以基于 [访问者模式 ](https://zh.wikipedia.org/wiki/%E8%AE%BF%E9%97%AE%E8%80%85%E6%A8%A1%E5%BC%8F)  来操作类，另一个是Tree API，以基于树节点来操作类。这一章我们讨论 Core API 。

在Core API中有以下几个关键类：

- **ClassReader：**这个类会将 .class 文件读入到 ClassReader 中的字节数组中，它的 accept 方法接受一个 ClassVisitor 实现类，并按照顺序调用 ClassVisitor 中的方法

- **ClassVisitor：**主要负责访问类的成员信息。包括标记在类上的注解、类的构造方法、类的字段、类的方法、静态代码块等。

- **ClassWriter：**ClassWriter 是一个 ClassVisitor 的子类，是和 ClassReader 对应的类，ClassReader 是将 .class 文件读入到一个字节数组中，ClassWriter 是将修改后的类的字节码内容以字节数组的形式输出。

![ASM里的核心类](https://lsieun.github.io/assets/images/java/asm/asm-core-classes.png)

这里我们先以一个生成全新字节码文件为例子：

```java
package asm.test1;

import org.objectweb.asm.*;
import java.io.*;

public class HelloWorld {
    public static void main(String[] args) throws Exception {
        byte[] bytes = generate();
        outputClazz(bytes);
    }
    private static byte[] generate() {
        ClassWriter classWriter = new ClassWriter(0);
        // 定义对象头；版本号、修饰符、全类名、签名、父类、实现的接口
        classWriter.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, "asm/test1/AsmHelloWorld", null, "java/lang/Object", null);
        // 添加方法；修饰符、方法名、描述符、签名、异常
        MethodVisitor methodVisitor = classWriter.visitMethod(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "main", "([Ljava/lang/String;)V", null, null);
        // 执行指令；获取静态属性
        methodVisitor.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        // 加载常量 load constant
        methodVisitor.visitLdcInsn("Hello World ASM!");
        // 调用方法
        methodVisitor.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
        // 返回
        methodVisitor.visitInsn(Opcodes.RETURN);
        // 设置操作数栈的深度和局部变量的大小
        methodVisitor.visitMaxs(2, 1);
        // 方法结束
        methodVisitor.visitEnd();
        // 类完成
        classWriter.visitEnd();
        // 生成字节数组
        return classWriter.toByteArray();
    }
    private static void outputClazz(byte[] bytes) throws Exception {
        // 输出类字节码
        String pathName = HelloWorld.class.getResource("").getPath() + "AsmHelloWorld.class";
        FileOutputStream out = new FileOutputStream(new File(pathName));
        System.out.println("ASM类输出路径：" + pathName);
        out.write(bytes);
    }
}
```

`generate()`方法使用ASM框架生成字节数组，`outputClazz()`将字节数组输出为`AsmHelloWorld.class`，在out的同级目录下：

![image-20220215212618111](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220215212618.png)

刚刚生成了字节码文件只用到了`ClassWriter`这一个类，因为我们并没有涉及到修改已有字节码文件的步骤，下一步我们要求在输出`Hello World ASM!`后，再输出一行`Hello World ASM Again!`。

回到最初介绍3个关键类的地方，**ClassReader** 接收一个 **ClassVisitor** 后，利用 `accept` 方法对 `.class` 类文件的内容从头到尾扫描一遍，每次扫描到类文件相应的内容时，都会调用**ClassVisitor**内部相应的方法。

- 扫描到**类文件**时，会回调`ClassVisitor`的`visit()`方法；
- 扫描到**类注解**时，会回调`ClassVisitor`的`visitAnnotation()`方法；
- 扫描到**类成员**时，会回调`ClassVisitor`的`visitField()`方法；
- 扫描到**类方法**时，会回调`ClassVisitor`的`visitMethod()`方法；

 ......

扫描到相应结构内容时，会回调相应方法，该方法会返回一个对应的字节码操作对象（比如，`visitMethod()`返回`MethodVisitor`实例），通过修改这个对象，就可以修改`class`文件相应结构部分内容，最后将这个`ClassVisitor`字节码内容覆盖原来`.class`文件就实现了类文件的代码切入。

概念可能有些晦涩，我们带入到上面的例子中理解，在扫描class文件时，有不同方法，我们需要找到main方法进入，所以在重写的`visitMethod()`中对方法名进行判断，返回一个自定义的`MethodVisitor`，在这个内部对原有代码进行修改。

```java
package asm.test1;

import org.objectweb.asm.*;
import java.io.*;

public class HelloWorldAgain {
    public static void main(String[] args)throws Exception {
        // 1. 创建 ClassReader 读取 .class 文件
        ClassReader classReader = new ClassReader("asm.test1.AsmHelloWorld");
        // 2. 创建 ClassWriter 对象，将操作之后的字节码的字节数组回写
        ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
        // 3. 创建自定义的 ClassVisitor 对象 classVisitor 需要委托 classWriter
        ClassVisitor classVisitor = new MyVisitor(classWriter);
        // 4. classReader 再委托给 classVisitor
        classReader.accept(classVisitor,classReader.EXPAND_FRAMES);

        byte[] bytes = classWriter.toByteArray();

        outputClazz(bytes);
    }
    private static void outputClazz(byte[] bytes) throws Exception {
        // 输出类字节码
        String pathName = HelloWorld.class.getResource("").getPath() + "AsmHelloWorld.class";
        FileOutputStream out = new FileOutputStream(new File(pathName));
        System.out.println("ASM类输出路径：" + pathName);
        out.write(bytes);
    }
    private static class  MyVisitor extends ClassVisitor{

        public MyVisitor(ClassVisitor classVisitor) {
            super(Opcodes.ASM9, classVisitor);
        }

        public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
//            System.out.println("=====================");
//            System.out.println("acce== " + access);
//            System.out.println("name== " + name);
//            System.out.println("desc== " + descriptor);
//            System.out.println("sign== " + signature);
//            System.out.println("=====================");
            MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
            // 进入main方法
            if (name.equals("main")){
                mv = new MyMethodVisitor(Opcodes.ASM9,mv);
            }
            return mv;
        }
    }
    private static class MyMethodVisitor extends  MethodVisitor{

        public MyMethodVisitor(int api, MethodVisitor methodVisitor) {
            super(api, methodVisitor);
        }
        @Override
        public void visitInsn(int opcode) {
  					// System.out.println(opcode);
            // 找到 return 指令，在 return 执行前插入代码
            if (opcode == Opcodes.RETURN) {
                hack(mv, "Hello World ASM Again!");
            }
            super.visitInsn(opcode);
        }

        private static void hack(MethodVisitor mv, String msg) {
            mv.visitFieldInsn(
                    Opcodes.GETSTATIC,
                    Type.getInternalName(System.class),
                    "out",
                    Type.getDescriptor(PrintStream.class)
            );
            mv.visitLdcInsn(msg);
            mv.visitMethodInsn(
                    Opcodes.INVOKEVIRTUAL,
                    Type.getInternalName(PrintStream.class),
                    "println",
                    "(Ljava/lang/String;)V",
                    false
            );
        }


    }
}
```

运行后，成功插入代码：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package asm.test1;

public class AsmHelloWorld {
    public static void main(String[] var0) {
        System.out.println("Hello World ASM!");
        System.out.println("Hello World ASM Again!");
    }
}
```

### Javassist

**ASM** 更偏向于底层，需要了解 **JVM** 虚拟机中指定规范以及对局部变量以及操作数栈的知识，相对而言**Javassist**操作使用上更加容易控制，虽然对对比上会比 **ASM** 性能差一些。

Javassist中最重要的是ClassPool、CtClass、CtMethod、CtField这四个类：

- **CtClass（compile-time class）**：编译时类信息，它是一个class文件在代码中的抽象表现形式，可以通过一个类的全限定名来获取一个CtClass对象，用来表示这个类文件。
- **ClassPool**：从开发视角来看，**ClassPool**是一张保存**CtClass**信息的**HashTable**，**key**为类名，**value**为类名对应的CtClass对象。当我们需要对某个类进行修改时，就是通过`pool.getCtClass("className")`方法从pool中获取到相应的CtClass。
- **CtMethod、CtField**：这两个比较好理解，对应的是类中的方法和属性。

```java
package javAssist.test2;

import javassist.*;

public class HelloWorld {
    public static void main(String[] args) throws Exception {

        ClassPool pool = ClassPool.getDefault();
        CtClass ctClass = pool.get("asm.test1.AsmHelloWorld");
        CtMethod ctMethod = ctClass.getDeclaredMethod("main");
        ctMethod.insertAfter("{System.out.println(\"javassist HelloWorld\");}");
        // 输出类内容
        ctClass.writeFile();
    }

}
```

成功插入：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package asm.test1;

public class AsmHelloWorld {
    public static void main(String[] var0) {
        System.out.println("Hello World ASM!");
        System.out.println("Hello World ASM Again!");
        Object var2 = null;
        System.out.println("javassist HelloWorld");
    }
}
```

简单的的确不是一星半点...

笔者认为相比于ASM，Javassist配合Agent技术，在链路监控，修改字节码等等方面可能会简便一些，但是如果涉及到一定规模上的字节码扫描，静态分析，那毫无疑问ASM会高效很多。

最后，本文也是抛砖引玉，站在很多前人的肩膀上的笔记，如果对ASM感兴趣的话，强推一波 [lsieun](https://lsieun.github.io/) 师傅的 [Java ASM 系列 ](https://lsieun.github.io/java/asm/index.html)，我就没见过能有这么详细的教程，真的良心。

## 后记 

### m1编译openjdk

写了这么多JVM，不自己编译一下也说不过去，目前大多数例子还是以Linux或者intelMac为主，也打算踩一下坑。

### 环境准备

- OS：Mac m1
- IDE：Clion
- 源码：OpenJDK8

#### 准备编译工具

```bash
🌀  c++  git clone https://github.com/AdoptOpenJDK/openjdk-jdk8u    
Cloning into 'openjdk-jdk8u'...
remote: Enumerating objects: 504871, done.
remote: Total 504871 (delta 0), reused 0 (delta 0), pack-reused 504871
Receiving objects: 100% (504871/504871), 1.00 GiB | 3.59 MiB/s, done.
Resolving deltas: 100% (416674/416674), done.
Updating files: 100% (47413/47413), done.
🌀  c++  ls
openjdk-jdk8u
🌀  c++  cd openjdk-jdk8u/       
🌀  openjdk-jdk8u [master] ls
ASSEMBLY_EXCEPTION THIRD_PARTY_README hotspot            make
LICENSE            common             jaxp               nashorn
Makefile           configure          jaxws              test
README             corba              jdk
README-builds.html get_source.sh      langtools

// 加速编译
brew install ccache  
// 字体引擎，编译过程中会被依赖到
brew install freetype 
brew install autoconf
```

下载Xcode，新版本在APP Store下载即可，低版本需要前往[苹果开发者网站](https://developer.apple.com/download/all/?q=xcode)。

#### 配置BOOT_JDK

我们编译jdk8，就需要准备一个低版本jdk7，其称为**BOOT_JDK**，这种用低版本编译高版本的方式也称为自举。

#### 安装Compiledb

需要python3+以及pip3+

```
pip install compiledb
```

#### 配置环境变量

```bash
# 设定语言选项，必须设置
export LANG=C
# Mac平台，C编译器不再是GCC，而是clang
export CC=clang
export CXX=clang++
export CXXFLAGS=-stdlib=libc++
# 是否使用clang，如果使用的是GCC编译，该选项应该设置为false
export USE_CLANG=true
# 跳过clang的一些严格的语法检查，不然会将N多的警告作为Error
export COMPILER_WARNINGS_FATAL=false
# 链接时使用的参数
export LFLAGS='-Xlinker -lstdc++'
# 使用64位数据模型
export LP64=1
# 告诉编译平台是64位，不然会按照32位来编译
export ARCH_DATA_MODEL=64
# 允许自动下载依赖
export ALLOW_DOWNLOADS=true
# 并行编译的线程数，编译时长，为了不影响其他工作，可以选择2
export HOTSPOT_BUILD_JOBS=4
export PARALLEL_COMPILE_JOBS=2 #ALT_PARALLEL_COMPILE_JOBS=2
# 是否跳过与先前版本的比较
export SKIP_COMPARE_IMAGES=true
# 是否使用预编译头文件，加快编译速度
export USE_PRECOMPILED_HEADER=true
# 是否使用增量编译
export INCREMENTAL_BUILD=true
# 编译内容
export BUILD_LANGTOOL=true
export BUILD_JAXP=true
export BUILD_JAXWS=true
export BUILD_CORBA=true
export BUILD_HOTSPOT=true
export BUILD_JDK=true
# 编译版本
export SKIP_DEBUG_BUILD=true
export SKIP_FASTDEBUG_BULID=false
export DEBUG_NAME=debug
# 避开javaws和浏览器Java插件之类部分的build
export BUILD_DEPLOY=false
export BUILD_INSTALL=false

# 最后需要干掉这两个环境变量（如果你配置过），不然会发生诡异的事件
unset JAVA_HOME
unset CLASSPATH

```

注意把当前java版本切换为BOOT_JDK

不切换也行，编译的时候需要加上参数

```bash
🌀  openjdk-jdk8u [master] ⚡  java -version
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)
```

#### 执行配置文件校验命令

```bash
sh configure --with-freetype-include=/opt/homebrew/opt/freetype/include/freetype2 --with-freetype-lib=/opt/homebrew/opt/freetype/lib --disable-zip-debug-info --disable-debug-symbols --with-debug-level=slowdebug --with-boot-jdk=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home --with-jvm-variants=server
```

上面代码只需要修改一下`--with-freetype-include`、`--with-freetype-lib`、`--with-boot-jdk`的位置

```
bash ./configure --with-debug-level=slowdebug --with-freetype-include=/usr/local/Cellar/freetype/2.10.4/include/freetype2 --with-freetype-lib=/usr/local/Cellar/freetype/2.10.4/lib/ --with-boot-jdk=/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home --enable-debug-symbols
```

**出现的几个问题：**

- `Xcode 4 is required to build JDK 8, the version found was 11.0.`

  修改	`common/autoconf/generated-configure.sh`，注释以下代码：

  ```sh
  XCODE_VERSION=`$XCODEBUILD -version | grep '^Xcode ' | sed 's/Xcode //'`
  XC_VERSION_PARTS=( ${XCODE_VERSION//./ } )
  if test ! "${XC_VERSION_PARTS[0]}" = "4"; then
    as_fn_error $? "Xcode 4 is required to build JDK 8, the version found was $XCODE_VERSION. Use --with-xcode-path to specify the location of Xcode 4 or make Xcode 4 active by using xcode-select." "$LINENO" 5
  fi
  ```

- 无法找到Xcode相关路径：

  ```
  checking Determining Xcode SDK path... xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
  ```

  执行：

  ```bash
  sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
  ```

- `configure: error: A gcc compiler is required. Try setting --with-tools-dir.`

  修改 `common/autoconf/generated-configure.sh`，注释以下代码，一共有两处

  ```bash
  $ECHO "$COMPILER_VERSION_OUTPUT" | $GREP "Free Software Foundation" > /dev/null
   #   if test $? -ne 0; then
   #     { $as_echo "$as_me:${as_lineno-$LINENO}: The $COMPILER_NAME compiler (located as $COMPILER) does not seem to be the required $TOOLCHAIN_TYPE compiler." >&5
  #$as_echo "$as_me: The $COMPILER_NAME compiler (located as $COMPILER) does not seem to be the required $TOOLCHAIN_TYPE compiler." >&6;}
   #     { $as_echo "$as_me:${as_lineno-$LINENO}: The result from running with --version was: \"$COMPILER_VERSION\"" >&5
  #$as_echo "$as_me: The result from running with --version was: \"$COMPILER_VERSION\"" >&6;}
    #    as_fn_error $? "A $TOOLCHAIN_TYPE compiler is required. Try setting --with-tools-dir." "$LINENO" 5
    #  fi
  ```

如果出现以下界面，则说明成功

![image-20220201122242240](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220201122242.png)

成功了个🔨，明明是64位，但是这里**OpenJDK target**却是检测的32位，我们修改一下config命令：

```bash
sh configure --with-freetype-include=/opt/homebrew/opt/freetype/include/freetype2 --with-freetype-lib=/opt/homebrew/opt/freetype/lib --disable-zip-debug-info --disable-debug-symbols --with-debug-level=slowdebug --with-boot-jdk=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home --with-jvm-variants=server --with-target-bits=64
```

- `configure: error: It is not possible to use --with-target-bits=64 on a 32 bit system.`

  新的报错，看来还是openjdk对arm系统不太兼容，我们在`common/autoconf/generated-configure.sh`重新搜索一下报错语句，强制加上

  `OPENJDK_TARGET_CPU_BITS=64`

终于ok了

![image-20220201123337750](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/02/20220201123337.png)

### 开始编译(放弃了)

执行`compiledb make WARNINGS_ARE_ERRORS="" CONF=macosx-arm-normal-server-slowdebug all `开始编译

注意：每次编译失败，都需要以下两个命令重新编译

1. `compiledb make CONF=macosx-arm-normal-server-slowdebug clean`
2. `compiledb make WARNINGS_ARE_ERRORS="" CONF=macosx-arm-normal-server-slowdebug all`

可能出现的问题：

- `ld: library not found for -lstdc++`

  下载`git clone https://github.com/quantum6/xcode-missing-libstdcpp`，进入工具执行`sh install.sh`，这是一个软连接命令，在macos高版本为了安全对软连接目录作了限制，我们可以对链接的文件目录做替换即可。

- ......

- ...... 

- 还有一大堆问题，m1，🐶都不用

## 参考

- [《 深入理解Java虚拟机 》](https://book.douban.com/subject/34907497/)
- [《 自己动手写Java虚拟机 》](https://book.douban.com/subject/26802084/)
- [Java虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se8/html/) 
-  [字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)
-  [fynch3r ASM笔记](https://fynch3r.github.io/ASM%E7%AC%94%E8%AE%B0/)


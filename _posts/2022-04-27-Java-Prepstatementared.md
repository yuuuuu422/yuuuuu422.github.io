---
title: 浅谈Java预编译
key: 浅谈Java预编译
---

之前面试问到了预编译相关的问题，感觉回答的不是很好，通过几个例子深入学习一下。

<!--more-->

Java预编译分服务端预编译和客户端预编译两种，对应的url参数为`useServerPrepStmts`，其为true时是在数据库服务端预编译，其为false则是在驱动包内进行的处理。

## 服务端预编译

先简单摘要一下数据库SQL语句的编译特性：

>数据库接受到sql语句之后，需要检查缓存、规则验证（**词法和语义解析**）、解析器解析为语法树、预处理器进一步验证语法树、优化SQL、生成执行计划、执行。这几个阶段和我们一些高级语言的解释执行差不多是一个道理。
>
>但很多时候，我们一条SQL语句可能会执行多次，每次执行可能只是个别的值不一样（比如query的where子句不一样），如果每次都经过上面重复的步骤，效率就会比较低了，因为对其中对语法的解析和优化的过程其实是与传入的字段值无关
>
>所以预编译使用占位符?代替字段值的部分，将SQL语句先交由数据库预处理，构建语法树，再传入真正的字段值多次执行，省却了重复解析和优化相同语法树的时间，提升了SQL执行的效率。

我们使用以下三条语句就可以简单在Mysql上使用预编译：
```sql
prepare stmt from 'select * from users where username = ?';
set @username="admin";
execute stmt using @username;
```

其对应的Java代码为：

![image-20220428005016462](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130250.png)

我们通过Wireshark可以发现的确是在服务端进行了处理：

![image-20220428005148389](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130252.png)

那如果我们传入的username带一个单引号，预编译会怎么处理呢？

![image-20220428005407244](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130253.png)

可以看到mysql服务端对我们传入的字段进行了转义，规避了单引号的闭合。不过说到底预编译的本身目的还是为了性能和效率，我认为其预防SQL注入只是处理时加上的一个特性而已（埋一个坑，有机会去看看Mysql源码）。

## 客户端预编译

在`connnect`连接时，不设置**useServerPrepStmts**（默认为false），则采用的是驱动包（**mysql-connector-java**）内的处理。

![image-20220428093236263](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130256.png)

![image-20220428094239596](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130300.png)

上图可以看到我们传入的`admin'`被处理为了`'admin'''`，最外层的单引号是本身会加上的，不过预处理把我们额外添加的单引号后又追加了一个单引号，规避了其闭合。同时注意一点wireshark抓包可见在服务端并没有 **Prepare**的处理。

我们在  `preparedStatement.setString(1,username);`处打下断点，定位到`ClientPreparedQueryBindings.setString`:

![image-20220428130231581](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130302.png)

逻辑其实很简单，先是对检查传入的字符串室友存在**Escaped**字符，如果存在则根据具体情况处理，最后再往前后添加单引号。其实简单来看也相当于是一个消毒处理。

之前说到客户端处的预编译并没有往服务端请求，相当于只是本地的一个缓存，那我们可以手动修改`SetValue `的值，来观察之后数据库的执行。

```java
public final synchronized void setValue(int paramIndex, byte[] val, MysqlType type) {
    this.bindValues[paramIndex].setByteValue(val);
    this.bindValues[paramIndex].setMysqlType(type);
}
```

从 **setValue** 来看我们最后设置的值存储在来 **bindValues** ，因为我们是通过`Class.forName`载入驱动包，所以需要通过反射修改其值，不过也可以在Debug的过程中用idea修改：

```txt
'' or 1=1 #
39 39 32 79 82 32 49 61 49 32 35 
```



![image-20220427214109417](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130305.png)

监控的执行情况：

![image-20220427214524712](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428130310.png)

发现数据库的确是执行了注入语句，打印了所有users信息。

抛开编译、AST等知识，站在安全的角度来说我认为预编译之所以能防御sql注入还是一个消毒函数的问题，本质上还是通过转义、追加等方式来规避单引号的闭合，从而让数据段和代码段不会混淆。

## 预编译所需要注意的几点

如果你在面试的时候简单说一句预编译，那伯分之伯还会追问一句使用预编译需要注意哪些地方，

### 可能出错的地方

这一部分主要是一些特殊的地方，预编译的写法需要注意。

#### like语句

以下两种写法都可以：

1. 使用`concat`拼接

```java
String sql = "select * from users where username like concat('%',?,'%')\n ";
PreparedStatement preparedStatement = connection.prepareStatement(sql);
preparedStatement.setString(1,"a");
ResultSet resultSet = preparedStatement.executeQuery();
```

2. 在setString中再添加

```java
String sql = "select * from users where username like ? ";
PreparedStatement preparedStatement = connection.prepareStatement(sql);
preparedStatement.setString(1,"%a%");
ResultSet resultSet = preparedStatement.executeQuery();
```

#### in语句

错误的写法：

```java
String ids = "1,2";
//String ids = "1,2) or 1=1#";
String sql = "select * from users where id in ("+ids+")";
Statement statement = connection.createStatement();
ResultSet resultSet=  statement.executeQuery(sql);
```

in语句的预编译，我们需要确定预编译的个数，这里采用分隔符的方法区分。

```java
StringBuilder temp = new StringBuilder();
String ids = "1,2";
// String ids = "1,2) or 1=1#,3";
String[]  splitIds = ids.split(",");
for(int i = 0; i< splitIds.length; i++){
    if(i == 0){
        temp.append("?");
    } else {
        temp.append(",?");
    }
}

String sql = "select * from users where id in ("+temp+")";
PreparedStatement preparedStatement = connection.prepareStatement(sql);

for(int i = 0; i < splitIds.length; i++){
    preparedStatement.setInt(i+1, Integer.parseInt(splitIds[i]));
}
ResultSet resultSet = preparedStatement.executeQuery();
```

### 无法使用预编译的场景

这一方面也同样可以通过服务端和客户端两个方面回答。

首先是服务端，看看尝试对查询的表名进行预编译：

`prepare stmt from 'select * from ? where username = ?';`

![image-20220428135058981](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428193632.png)

发现报错了，其实很好理解，**表名和列名是不能够被预编译的**，因为生成语法树的过程中，预处理器在进一步检查解析后的语法树时，**检查数据表和数据列是否存在**，如果这两个值是占位符`	?`所代替，自然会报错。

从驱动包的角度，之前所说java预编译会在占位符前后自动添加两个单引号，那么如果我们执行以下的语句：

![image-20220428135913397](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428193629.png)

实际上执行的语句为：

![image-20220428140036366](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2022/04/20220428193627.png)

Mysql表名不允许使用单引号，所以会报语法错误。

那列名呢，当然也是不行，比如`select 'username' from users`，这里会把`'username'`当作一个具体的值而不是列名，从而导致执行结果不一致。

直接增删改查的表名列名不行，同理像类似order by的地方，需要用到列名来排序的地方同样不行。

针对上述几种情况，比较好的方式还是使用白名单，毕竟表名和列名大多时候是我们提前确定的值。

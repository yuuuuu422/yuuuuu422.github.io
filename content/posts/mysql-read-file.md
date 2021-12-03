---
date: 2021-08-08T1:20:20+08:00
title :	When Mysql read your file
---



![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210808211512.png)

这并不是一个新鲜的漏洞，准确的来说是炒冷饭，不过刚好学了计网，就来分析一下吧，顺便用go来重写一个简易蜜罐。

我们先看一看mysql客户端连接的这个过程，进行了怎样的通讯。

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210808200246.png)

我们无需在意tcp的连接过程，所以直接过滤mysql协议即可。总的来说这里经历了5个过程。

1. S➡C `Server Greeting`
2. C➡S `Login Request`
3. S➡C `Response OK`
4. C➡S`Requst Query`
5. S➡C`Respose`

当客户端连接上服务器，服务器就会发送**Server Greeting**这第一个握手数据包，这些数据的内容取决于服务器版本和服务器配置

```
0000   4a 00 00 00 0a 35 2e 35 2e 35 33 00 05 00 00 00   J....5.5.53.....
0010   31 47 50 3d 22 31 6e 58 00 ff f7 21 02 00 0f 80   1GP="1nX...!....
0020   15 00 00 00 00 00 00 00 00 00 00 4d 5b 5e 48 2e   ...........M[^H.
0030   49 67 4b 7e 5a 27 52 00 6d 79 73 71 6c 5f 6e 61   IgK~Z'R.mysql_na
0040   74 69 76 65 5f 70 61 73 73 77 6f 72 64 00         tive_password.
```

我们可以在[MySQL官方文档](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::Handshake)中探寻这一些数据所对应的含义，不过从蜜罐的角度，我们只要在接收连接时，原封不动把这个包发过去即可。

第二个包，也就是客户端的登陆包了，里面带着user信息和密码

```
0000   50 00 00 01 *85* a6 ff 20 00 00 00 01 21 00 00 00   P...... ....!...
0010   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0020   00 00 00 00 72 6f 6f 74 00 14 2c 32 07 d2 b5 a0   ....root..,2....
0030   ff d5 6b 26 1f 09 83 5c 0e 4e 42 e5 da ce 6d 79   ..k&...\.NB...my
0040   73 71 6c 5f 6e 61 74 69 76 65 5f 70 61 73 73 77   sql_native_passw
0050   6f 72 64 00                                       ord.
```

这个包本身没有什么好说的，但是我们重点看第五个字节：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210808200307.png)

这个flag为**Can Use LOAD DATE LOCAL**,value为1的话，表示允许客户端读取本地文件，关于这个我们后面再讨论。

第三个包也就是`Response OK`,原封不动照搬即可。

```
0000   07 00 00 02 00 00 00 02 00 00 00                  ...........
```

第四个包是客户端的一个请求包，mysql连接后客户端会**默认**向服务端发起一个`select @@version`的请求来获取版本信息，第五个包也就是服务端返回的信息。

这样来看好像还没有走到漏洞点，现在通过`load data local`来看一看这个过程：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210808200321.png)

第一个包**Request Query**：`load data local infile "/etc/passwd" into table test;`也就是客户端发起的请求，意味把自己的`/etc/passwd`读取到test表中，而客户端是否有这个权限也就在于之前说的**Can Use LOAD DATE LOCAL**是否为1，其中**windows和Linux**默认为1，**mac**默认为0。

第二个包**Response TABULAR**是服务端的返回包,确认客户端应该发送的内容。

```
0000   0c 00 00 01 fb 2f 65 74 63 2f 70 61 73 73 77 64   ...../etc/passwd
```

其中`00 00 01 fb`是初始头，fb后为文件。

第三个包是客户端返回的文件信息：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/08/20210808200334.png)

其中前4个字节是我们不需要的，前三个为数据长度，第四个为序列号。

第四个是服务端返回的确认信息，这个无所谓，因为数据已经拿到了。

看到这，你应该知道是怎么回事了，正是上面**Response TABULAR**这一过程，让服务端有了伪造文件读取的机会，官网也是这么表示这一过程：

>In theory, a patched server could be built that would tell the client  program to transfer a file of the server's choosing rather than the file named by the client in the LOAD DATA statement

用lightless的表述也就是：

- 客户端：hi~ 我将把我的 AAA文件给你插入到 test 表中！
- 服务端：OK，读取你本地的 BBB文件并发给我！
- 客户端：这是文件内容：balabal（BBB文件的内容）！

并且，事实上并不是一定要客户端发起**Load local file**后服务端才能修改**Response TABULAR**，只要是客户端的**Request Query**,我们都可以发起**Response TABULAR**去进行文件读取，而正如最开始所说的，客户端连接后会发起一个`select @@version`,我们可以紧随以后构造请求。

代码请见→[github](https://github.com/yuuuuu422/Go-Toys/tree/main/evil-mysql)

防护：使用`--ssl-mode=VERIFY_IDENTITY`来建立可信的连接。但事实上作为蜜罐的话，大多扫描器为了效率并不会使用ssl。

参考：

- [Read MySQL Client's File -lightless ](https://lightless.me/archives/read-mysql-client-file.html)
- [[MySQL connect file read](http://russiansecurity.expert/2016/04/20/mysql-connect-file-read/)]
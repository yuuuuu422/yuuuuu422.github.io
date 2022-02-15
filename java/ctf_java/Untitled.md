环境说明：

java题很多时候是直接给一个jar包，可以通过idea反编译看源码

![D1479182-A878-4809-B7FA-6AE5E3AB9B6C](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221145711.png)

但是springboot打包的项目，加入到library之后通常是不可以直接通过import导入其中的package，这里我是把jar包解压，然后把BOOT-INF里的class文件提取出来，放在一个专门的lib文件夹下重新导入library。

![image-20211221150135789](https://cdn.jsdelivr.net/gh/yuuuuu422/Myimages/img/2021/12/20211221150135.png)

本地调试的话，还是直接运行jar包即可，不过如果需要写poc，就可以在src下导入原文件进行编写。
---
title: 深入学习OpenRASP
key:  LearnRASP
---

## 环境准备

### 安装elasticsearch

rasp_cload要求elasticsearch版本5.6和7.0之间

>[*es.go*:*65*]  [*30003*] unable to support the ElasticSearch with a version greater than or equal to *7*.*0*.*0*, the current version is *7*.*17*.*1*

brew下载默认在7.0.0以上，需要去官网下载低版本，之后运行：

```bash
sh ./bin/elasticsearch
```

访问http://localhost:9200/

```json
{
  "name" : "JCQtDvA",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "pPpZmDlJQLOFJwheC-gc3A",
  "version" : {
    "number" : "6.8.23",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "4f67856",
    "build_date" : "2022-01-06T21:30:50.087716Z",
    "build_snapshot" : false,
    "lucene_version" : "7.7.3",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

### 安装mangoDB

```
brew tap mongodb/brew
```

 >MongoDB 5.0 社区版删除了对 macOS 10.13 的支持

```
brew install mongodb-community@5.0
```

启动和关闭：

```
brew services start mongodb-community@5.0
brew services stop mongodb-community@5.0
```

访问 http://127.0.0.1:27017/

```txt
It looks like you are trying to access MongoDB over HTTP on the native driver port.
```

### 安装后台

安装[rasp-cloud-mac.tar.gz](https://github.com/baidu/openrasp/releases/download/v1.3.7/rasp-cloud-mac.tar.gz)，执行`./rasp-cloud -d`

### 安装agent

```bash
java -jar RaspInstall.jar -heartbeat 90 -appid d9f8607e2392af31c0983ed09a4d725dea874c43 -appsecret wovX05qFLLdBccN84hP1ohKzEFXGbfBt3QJj3L9uPya -backendurl http://127.0.0.1:8086/ -nodetect -install /Users/theoyu/workspace/java/vul_samples/java-sec-code
```

### 主机上线

![image-20220314141845416](../../../../Library/Application Support/typora-user-images/image-20220314141845416.png)

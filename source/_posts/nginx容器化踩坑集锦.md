---
title: nginx容器化踩坑集锦
date: 2024-03-13 21:28:12
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

在容器内一顿操作后使用curl命令查看配置是否成功，结果返回错误代码。这时候我们可以通过错误代码来分析是哪配置出错了，但是最简单快捷的方法是直接查看nginx的错误日志。

## 如何查看容器内nginx的日志

nginx的日志默认存放在：

/var/log/nginx/access.log
/var/log/nginx/error.log

但是我们使用cat命令无法查看日志的内容

```
root@791005f0c7ca:/etc/nginx# ls -la /var/log/nginx/
total 8
drwxr-xr-x 2 root root 4096 Feb 14 19:53 .
drwxr-xr-x 1 root root 4096 Feb 14 19:53 ..
lrwxrwxrwx 1 root root   11 Feb 14 19:53 access.log -> /dev/stdout
lrwxrwxrwx 1 root root   11 Feb 14 19:53 error.log -> /dev/stderr
```

我们通过命令查询后发现日志文件被重定向到标准输出和标准错误

`docker logs` 命令用于获取容器的日志信息。它的工作原理如下：

1. **查询容器日志文件位置**：Docker Daemon（dockerd）负责管理容器的标准输出和标准错误。当你运行容器时，你可以通过重定向或者Dockerfile中的指令将容器的日志输出到标准输出（stdout）或者标准错误（stderr）。`docker logs` 命令会检查容器的日志配置，并根据配置找到相应的日志文件路径。
2. **读取容器日志文件**：一旦确定了容器的日志文件位置，`docker logs` 命令就会读取这些日志文件的内容。它会将日志文件的内容输出到标准输出（stdout），从而在终端或者其他输出设备上显示。
3. **输出日志内容**：`docker logs` 命令会将读取到的日志内容输出到终端上，让用户可以查看容器的日志信息。这包括了容器启动时输出的信息，以及在运行过程中写入到日志文件的内容。

## 403 无权限

default.conf

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

书接上文，当我们配置好default.conf后，使用curl命令查看配置是否正确：

```
$ curl http://127.0.0.1:90/mock
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.25.4</center>
</body>
</html>
```

查看nginx容器的错误日志：

```
$ docker logs [container id]
[error] 233#233: *8 open() "/root/dist" failed (13: Permission denied), client: 172.17.0.1, server: , request: "GET /mock HTTP/1.1", host: "127.0.0.1:90"
```

我们发现nginx没有该文件夹的访问权限

于是使用常规操作，在nginx容器内给该文件夹提高权限：

```
$ chmod -R 644 /root/dist
```

提权后访问仍然出错，我突然发现我只给了dist文件夹权限，但是默认的nginx用户进不了root文件夹

解决方法就是将其移动到其他目录下，或者将nginx.conf中的user改为root

## 301

我们修改完上一个错误后，重新使用curl来验证：

```
$ curl http://127.0.0.1:90/mock
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.25.4</center>
</body>
</html>
```

出现了301错误，我参考[这篇文章](https://juejin.cn/post/7021818339651485726)做出了修改：

> 301是永久重定向。如果使用Nginx作为HTTP服务器，那么当用户输入一个不存在的地址之后，基本上会有两种情况：1.返回404状态码，2.返回301状态码和重定向地址。

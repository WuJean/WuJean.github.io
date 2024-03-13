---
title: Docker-多容器通信实战-（node+mongDB+nginx）
date: 2024-03-12 19:49:37
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

最近刚好在学nginx，mongdb和docker，想找一个项目练练手部署一下项目，巩固一下知识。在掘金上找到了一篇使用容器部署 [api-mocker](https://link.segmentfault.com/?enc=EEDlAqeJUvpyDp76PwLLxw%3D%3D.%2FrtZZ9Eecndm4LTwdqKVrqnBCdyL7OnmkjQRAYR0vcLMwiESEJmCeUuHykgQFJ5H)项目的博客，于是决定跟着大佬的教程做一遍，对学习的知识进行查缺补漏。

## 项目结构

该项目分为三个部分，需要建立三个容器（node+mongDB+nginx），各个容器之间使用docker的link参数进行相互通信。Docker 在版本 1.9 之后已经弃用了 `--link` 参数，而是推荐使用 Docker 网络（Docker network）来连接容器。通过 Docker 网络，你可以创建一个私有网络，并在需要的容器之间建立通信。

我们分别采用两种方法进行本次的部署。

## 实现过程

1、构建mongo容器
2、构建node容器并与mongo容器建立连接
3、构建nginx容器并与node容器建立连接

### 构建mongo容器

```crystal
docker pull mongo:latest
```

```
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
mongo         latest    79112eff9c89   11 days ago     756MB
```

用命令运行mongdb容器：

```
$ docker run --name mock-mongo -d -p 27017:27017 mongo:latest --auth
2ac9f5613c75f050f9aede527e1bcd211ece88b3dbae29c39134ddf00f74a743
```

- `docker run`: 这是 Docker 命令的一部分，用于启动新的容器。
- `--name mock-mongo`: 通过 `--name` 参数指定容器的名称，这里设置为 `mock-mongo`。
- `-d`: 通过 `-d` 参数让容器在后台以守护进程（detached mode）的形式运行，这样你可以在终端继续执行其他命令，而不会被容器的输出打扰。
- `-p 27017:27017`: 通过 `-p` 参数映射容器的端口到主机上，这里将容器的 `27017` 端口映射到主机的 `27017` 端口上。这意味着你可以通过主机的 `27017` 端口访问 MongoDB 容器。
- `mongo:latest`: 这是要运行的容器的镜像名称和标签。在这里，我们使用了 `mongo:latest`，它表示最新版本的 MongoDB 官方镜像。Docker 将从 Docker Hub 下载此镜像（如果本地没有的话）并启动一个容器。
- `--auth`: 这是传递给 MongoDB 容器的参数，它表示启用身份验证（authentication）。在 MongoDB 中启用身份验证可以增强安全性，因为用户需要提供有效的凭据才能访问数据库。注意，这个参数应该放在镜像名称和标签之后，这样 Docker 才能正确解析它，并将其传递给 MongoDB 容器。

查看容器mongo的运行状态：

```
$ docker ps -a
CONTAINER ID   IMAGE          COMMAND                   CREATED              STATUS                    PORTS                                           NAMES
2ac9f5613c75   mongo:latest   "docker-entrypoint.s…"   About a minute ago   Up About a minute         0.0.0.0:27017->27017/tcp, :::27017->27017/tcp   mock-mongo
```

进入容器编辑一下配置文件：

```
$ docker exec -it mock-mongo /bin/bash
```

编辑node连接时使用的账号：

```
root@2ac9f5613c75:/usr/bin# mongosh admin
Current Mongosh Log ID: 65f04c44b156da9055c1566f
Connecting to:          mongodb://127.0.0.1:27017/admin?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.1.5
Using MongoDB:          7.0.6
Using Mongosh:          2.1.5

For mongosh info see: https://docs.mongodb.com/mongodb-shell/


To help improve our products, anonymous usage data is collected and sent to MongoDB periodically (https://www.mongodb.com/legal/privacy-policy).
You can opt-out by running the disableTelemetry() command.

admin> db.createUser({
...     user: "admin",
...     pwd: "admin",
...     roles: [{ role: "userAdminAnyDatabase", db: "admin" }]
... })
{ ok: 1 }
admin> db.auth('admin','admin')
{ ok: 1 }
```

## 构建node容器并与mongo建立连接

由于api-mocker有进行更新迭代，我们要选择好分支才能修改mongo的连接配置：

```
$ git checkout master
分支 'master' 设置为跟踪来自 'origin' 的远程分支 'master'。
切换到一个新分支 'master'
$ vi server/config/config.default.js
```

```
  mongoose: {
    url: 'mongodb://admin:admin@db:27017/api-mock?authSource=admin'
  },
```

接着我们编写一个Dockerfile文件来构建镜像：

```
# 指定基础镜像
FROM node:latest

# 维护者
MAINTAINER wujean1220@gmail.com

# 工作目录
WORKDIR /www

# 将本地项目拷贝到容器中 不解压
COPY api-mocker node-server/api-mocker

EXPOSE 7001

WORKDIR /www/node-server/api-mocker/server

RUN npm install

WORKDIR /www/node-server/api-mocker

## 构建容器之后 容器启动时执行的命令
CMD ["make", "prod_server"]
```

开始构建容器：

```
$ docker build -t="mock-server:1.0.0" .
```

```
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
mock-server   1.0.0     49a559bb7dca   18 seconds ago   1.44GB
```

将mock-server容器运行起来并建立网络通信：

```
$ docker run -d -i -t -p 7001:7001 --name mock-server1 --link mock-mongo:db mock-server:1.0.0 /bin/bash
11f08a750decfd94a54200dd5107621bb92c687ba7165c882872f31d8c3011e3
$ docker ps
CONTAINER ID   IMAGE               COMMAND                   CREATED          STATUS          PORTS                                           NAMES
11f08a750dec   mock-server:1.0.0   "docker-entrypoint.s…"   14 seconds ago   Up 13 seconds   0.0.0.0:7001->7001/tcp, :::7001->7001/tcp       mock-server1
2ac9f5613c75   mongo:latest        "docker-entrypoint.s…"   38 minutes ago   Up 38 minutes   0.0.0.0:27017->27017/tcp, :::27017->27017/tcp   mock-mongo
```

检测node容器和mongo容器的连接状态：

```
 $ docker exec -it mock-server /bin/bash
 $ curl -p 27017 db
```

由于node容器里面没有ping命令，且curl是通过http协议来传输的，我们需要使用telnet命令来查看端口的情况。

## 构建nginx容器并与node容器建立连接

在建立nginx前，我们要先约定好，node容器别名，nginx转发的端口号以及客户端访问nginx域名及端口号

- node服务器别名：node
- node容器映射的端口号：7001
- nginx域名：127.0.0.1
- nginx端口号：90

拉取nginx容器并进行配置：

```
$ docker pull nginx:latest
$ docker run -p 90:80 --link mock-node:node nginx:latest --name mock-nginx
# 查看容器连接状态
$ docker exec -it mock-nginx /bin/bash
$ env
# 看到以下数据则表示连接成功了
NODE_PORT_7001_TCP=tcp://172.17.0.3:7001
NODE_PORT_7001_TCP_PORT=7001
NODE_ENV_YARN_VERSION=1.9.4
```

我们现在需要修改nginx的配置：

1. **nginx.conf**：
   - `nginx.conf`是Nginx服务器的主要配置文件，其中包含了全局配置指令和默认值，如`worker_processes`、`pid`等。
   - 除了全局配置指令，`nginx.conf`文件还可能包含其他配置文件，如`default.conf`等，以及其他模块的配置文件，如`http`、`mail`、`events`等。
   - 在`nginx.conf`中，还可以包含`include`指令来引入其他配置文件，使得配置文件的管理更加灵活。
2. **default.conf**：
   - `default.conf`是Nginx默认虚拟主机的配置文件。
   - 当在Nginx服务器上添加新的虚拟主机时，每个虚拟主机通常都会有一个独立的配置文件，但如果没有为特定虚拟主机指定配置文件，Nginx将使用`default.conf`作为默认配置。
   - `default.conf`文件通常包含一个`server`块，该块定义了Nginx服务器默认虚拟主机的一些基本配置，如监听的端口和服务器名称等。
   - 用户可以根据需要修改`default.conf`文件来自定义默认虚拟主机的行为，或者创建新的虚拟主机配置文件。

由于nginx的docker容器没有vi指令，我们使用docker cp命令在本机修改好文件后再上传到容器中：

```
$ docker cp mock-nginx:/etc/nginx/conf.d/default.conf ~/nginx/default.conf
$ vi ~/nginx/default.conf

server {
   location /mock-api/ {
       # node 为指令服务端容器别名
       proxy_pass http://172.17.0.3:7001/;
   }

   location /mock {
     autoindex on;
     alias /root/dist/;
   }
}

$ docker cp ~/nginx/default.conf mock-nginx:/etc/nginx/conf.d/default.conf
```

```
$ docker cp mock-nginx:/etc/nginx/nginx.conf ~/nginx/nginx.conf
$ vi nginx.conf 

我们需要将user从nginx改为root

$ docker cp ~/nginx/nginx.conf mock-nginx:/etc/nginx/nginx.conf
```

进入容器内重新加载nginx配置：

```
$ docker exec -it mock-nginx /bin/bash
$ docker exec -it mock-nginx /bin/bash
```

在项目的客户端下使用npm install && npm build 生成请求文件并上传到nginx配置的/root/dist目录下

在mock-server1中的server中启动服务端项目

访问前端项目： [http://127.0.0.1](https://link.segmentfault.com/?enc=ufPAyUllPWWcxr9CC8PTrg%3D%3D.XSEwDFoPajkmgBlrNfMHUiWfWRc4j643CWYKJm%2Feymg%3D):90/mock 成功出现页面完成配置。



在配置nginx容器中踩了许多坑，如果你也遇到问题，请移步 nginx容器化踩坑集锦

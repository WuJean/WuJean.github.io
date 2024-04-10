---
title: 腾讯云服务器配置clash科学上网
date: 2024-04-08 22:22:36
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover: https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20240410204314693.png
---

最近要在云服务上做一个项目，github上一直拉不下来代码，由于安装的是无图形化界面的linux系统，故记录一下在云服务器上配置clash的过程。

## 下载Clash

由于众多clash的版本都删库跑路了，于是找到第三方保存下来的包：https://www.clash.la/releases/

选择符合自己系统版本的clash：

```
wget https://down.clash.la/Clash/Core/Premium/clash-linux-amd64-latest.gz
```

或者使用远程scp命令上传到服务器上再解压

```
gzip -d clash-linux-amd64-latest.gz
mv clash-linux-amd64-latest clash
```

## 配置Clash

clash需要两个配置文件，分别为Country.mmdb和config.yaml

> Country.mmdb 文件是 Clash 配置文件中的一个数据库文件，它包含了 IP 地址对应的国家或地区信息。具体来说，".mmdb" 文件是 MaxMind 公司提供的 GeoIP2 数据库文件的一种格式，用于将 IP 地址映射到对应的地理位置信息，如国家、地区、城市等。这样的文件在网络代理工具中被用来进行基于地理位置的策略选择或者限制。

我们在文件夹中执行

```
./clash -d .
```

会自动生成Country.mmdb和默认的config.yaml，我们在机场里下载的文件就是配置文件yaml，其中包含了代理服务器的各种有关信息和自定义的分流规则，我们在生成yaml后只需要把我们本机上的yaml复制到服务器上。

```
# HTTP 代理端口
port: 7890 
 
# SOCKS5 代理端口
socks-port: 7891 
 
# Linux 和 macOS 的 redir 代理端口
redir-port: 7892 
 
# 允许局域网的连接
allow-lan: true
 
# 规则模式：Rule（规则） / Global（全局代理）/ Direct（全局直连）
mode: rule
 
# 设置日志输出级别 (默认级别：silent，即不输出任何内容，以避免因日志内容过大而导致程序内存溢出）。
# 5 个级别：silent / info / warning / error / debug。级别越高日志输出量越大，越倾向于调试，若需要请自行开启。
log-level: silent
# Clash 的 RESTful API
external-controller: '0.0.0.0:9090'
 
# RESTful API 的口令
secret: '123456' 
```

再次启动clash

```
./clash -d .
```

看到配置加载成功即可

![image-20240410204314693](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20240410204314693.png)

此时我们打开另一个终端，设置终端代理

```
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

尝试一下终端是否可以访问外网

```
wget www.youtube.com
```

## 将 Clash注册为系统服务并自启动

将 Clash移动到`/opt`文件夹下，`/opt` 文件夹通常用于存放可选的或者第三方安装的软件或程序

```
sudo mkdir -p /opt/clash
sudo mv clash /opt/clash
```

将配置文件移动到`/etc`文件夹下，`/etc` 文件夹在 Unix 和 Linux 系统中是一个非常重要的目录，通常用于存放系统的配置文件和配置相关的数据。"etc" 在这里代表 "et cetera"，意思是 "等等" 或者 "其它"，表明该目录中包含了系统中各种各样的配置文件。

```
sudo mkdir -p /etc/clash
sudo mv Country.mmdb /etc/clash
sudo mv config.yaml /etc/clash
```

### 设置为系统服务

systemd 是一种系统和服务管理器，用于大多数现代 Linux 发行版中。它负责启动系统中的各种服务、进程和任务，并管理它们的生命周期。systemd 使用单个配置文件来定义服务的行为和属性，通常位于 `/etc/systemd/system/` 目录下，以 `.service` 扩展名结尾。

使用 `vi` 增加 systemd 配置 `sudo vi /etc/systemd/system/clash.service` 放入如下内容

```
[Unit]
Description=Clash Service
After=network.target
 
[Service]
Type=simple
User=root
ExecStart=/opt/clash/clash -d /etc/clash
Restart=always
 
[Install]
WantedBy=multi-user.target
```

- `[Unit]`：描述了服务的基本信息和依赖关系。

  - `Description=Clash Service`：指定服务的描述为 "Clash Service"，这个描述将在启动服务时显示。

  - `After=network.target`：指定服务应该在网络已经启动之后才启动，确保网络环境已经准备好。

- `[Service]`：定义了服务的执行方式和行为。

  - `Type=simple`：指定服务类型为简单类型，表示 systemd 不会等待服务启动完成，而是立即认为服务已启动。

  - `User=root`：指定服务运行的用户为 root 用户。这意味着该服务将以 root 用户的权限运行。

  - `ExecStart=/opt/clash/clash -d /etc/clash`：指定服务启动时执行的命令。在这里，服务将运行 `/opt/clash/clash` 这个可执行文件，并传递 `-d /etc/clash` 参数。这个参数通常用来指定 Clash 的配置文件路径。

  - `Restart=always`：指定当服务意外终止时，systemd 将自动重新启动服务。

- `[Install]`：定义了安装服务的信息。
  - `WantedBy=multi-user.target`：指定服务应该在多用户模式下启动。这意味着系统启动到多用户模式时，该服务将自动启动。

### 启动系统服务

```
# 重载服务 
sudo systemctl daemon-reload 
# 开机启动 
sudo systemctl enable clash.service 
# 启动服务 
sudo systemctl start clash.service
# 查看服务状态 
sudo systemctl status clash.service
```

### 配置永久的系统代理

```
# 打开bashrc
vim ~/.bashrc
# 最底下写入
export http_proxy=127.0.0.1:7890
export https_proxy=127.0.0.1:7890

source ./.bashrc
```

## Clash的配置文件和修改

这段文本是一个配置文件，用于配置代理服务器的设置、代理规则、代理组等。下面是每个部分的详细解释：

1. `port: 7890`：指定 HTTP 代理服务器监听的端口号为 7890。
2. `socks-port: 7891`：指定 SOCKS5 代理服务器监听的端口号为 7891。
3. `redir-port: 7892`：指定用于 Linux 和 macOS 的 redir 代理的端口号为 7892。
4. `allow-lan: true`：允许局域网内的设备连接代理服务器。
5. `mode: Rule`：规定代理模式为 Rule（规则模式），即根据规则进行代理。
6. `log-level: silent`：设置日志输出级别为 silent，即不输出任何日志内容。
7. `external-controller: '0.0.0.0:9090'`：指定 Clash 的 RESTful API 监听的地址和端口为 0.0.0.0:9090。
8. `secret: ''`：RESTful API 的口令为空。
9. `proxies`：定义了一系列的代理服务器配置，包括名称、类型、服务器地址、端口、加密方式、密码等信息。
10. `proxy-groups`：定义了一系列代理组，包括自动选择、手动选择、国内、国际媒体、其他等不同类型的代理组。
11. `rules`：定义了一系列的代理规则，包括域名、IP、关键词等，指定了对应的代理行为。例如，将一些国内网站直连（Domestic）、一些国际媒体网站使用代理（Global_media）、一些特定网站使用代理等。

12. `Rule`（规则）部分：
    - 指定了具体的代理规则，以实现不同网站、IP地址或关键词的不同处理方式。例如，对于一些国内网站，使用直连（DIRECT），而对于一些国际网站或特定IP地址，使用代理服务器进行访问。
    - 这里的规则可以通过关键词匹配、正则表达式、IP地址等方式进行定义。

13. `Proxy`（代理）部分：
    - 这一部分定义了一系列的代理服务器配置，包括名称、类型、服务器地址、端口、密码、加密方式等信息。
    - 每个代理服务器配置包括了一个唯一的名称，以及相应的服务器信息，例如服务器地址、端口号、连接密码等。

14. `Proxy Group`（代理组）部分：
    - 这部分定义了一系列的代理组，用于将多个代理服务器进行组合，以实现更灵活的代理策略。
    - 每个代理组包括了一个唯一的名称，以及组内所包含的代理服务器列表和相应的策略（例如选择最快的服务器、随机选择等）。

15. `Rule`（规则）部分：
    - 规则部分定义了一系列的代理规则，以决定不同网站、IP地址或关键词的访问方式。
    - 每条规则包括了匹配条件和对应的处理方式，例如对某些网站进行直连、对某些网站使用特定的代理服务器等。

这些部分共同构成了一个完整的代理服务器配置文件，用于配置代理服务器的工作方式、代理规则和代理组等信息，以实现对网络流量的灵活控制和管理。

---
title: 拿到一台新的ubuntu机子后可能要做的事情
date: 2024-01-03 14:34:03
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

# 安装conda

```text
wget https://mirrors.bfsu.edu.cn/anaconda/archive/Anaconda3-2022.10-Linux-x86_64.sh --no-check-certificate
```

```text
bash Anaconda3-2021.11-Linux-x86_64.sh
```

写入配置文件

```
export PATH="~/anaconda3/bin":$PATH
source ~/anaconda3/bin/activate
```

```
source ~/.bashrc
```

# 安装docker

### 卸载旧版本

旧版本的 Docker 称为 `docker` 或者 `docker-engine`，使用以下命令卸载旧版本：

```
sudo apt-get remove docker \
docker-engine \
docker.io
```

## 使用 APT 安装

由于 `apt` 源使用 HTTPS 以确保软件下载过程中不被篡改。因此，我们首先需要添加使用 HTTPS 传输的软件包以及 CA 证书。

```
sudo apt-get update

sudo apt-get install \
apt-transport-https \
ca-certificates \
curl \
gnupg \
lsb-release
```

鉴于国内网络问题，强烈建议使用国内源，官方源请在注释中查看。

为了确认所下载软件包的合法性，需要添加软件源的 `GPG` 密钥。

```
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

> 以上命令会添加稳定版本的 Docker APT 镜像源，如果需要测试版本的 Docker 请将 stable 改为 test。

### 安装 Docker

更新 apt 软件包缓存，并安装 `docker-ce`：

```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

## 使用脚本自动安装

在测试或开发环境中 Docker 官方为了简化安装流程，提供了一套便捷的安装脚本，Ubuntu 系统上可以使用这套脚本安装，另外可以通过 `--mirror` 选项使用国内源进行安装：

> 若你想安装测试版的 Docker, 请从 test.docker.com 获取脚本

```
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh --mirror Aliyun
```

执行这个命令后，脚本就会自动的将一切准备工作做好，并且把 Docker 的稳定(stable)版本安装在系统中。

## 启动 Docker

```
sudo systemctl enable docker
sudo systemctl start docker
```

## 建立 docker 用户组

默认情况下，`docker` 命令会使用 [Unix socket](https://en.wikipedia.org/wiki/Unix_domain_socket) 与 Docker 引擎通讯。而只有 `root` 用户和 `docker` 组的用户才可以访问 Docker 引擎的 Unix socket。出于安全考虑，一般 Linux 系统上不会直接使用 `root` 用户。因此，更好地做法是将需要使用 `docker` 的用户加入 `docker` 用户组。

建立 `docker` 组：

```
sudo groupadd docker
```

将当前用户加入 `docker` 组：

```
sudo usermod -aG docker $USER
```

退出当前终端并重新登录，进行如下测试。

## 测试 Docker 是否安装正确

```
docker run --rm hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete
Digest: sha256:308866a43596e83578c7dfa15e27a73011bdd402185a84c5cd7f32a88b501a24
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

若能正常输出以上信息，则说明安装成功。

## 镜像加速

```
sudo vi /etc/docker/daemon.json
```

写入

```
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

重启服务：

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# 安装gpu驱动

```
ubuntu-drivers devices
```

```
== /sys/devices/pci0000:c2/0000:c2:02.0/0000:c3:00.0 ==
modalias : pci:v000010DEd00002204sv000010DEsd00001454bc03sc00i00
vendor   : NVIDIA Corporation
driver   : nvidia-driver-525-open - distro non-free
driver   : nvidia-driver-535-server-open - distro non-free
driver   : nvidia-driver-525-server - distro non-free
driver   : nvidia-driver-525 - distro non-free
driver   : nvidia-driver-535-open - distro non-free
driver   : nvidia-driver-470 - distro non-free
driver   : nvidia-driver-535 - distro non-free recommended
driver   : nvidia-driver-470-server - distro non-free
driver   : nvidia-driver-535-server - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

选择第一个安装：

```
sudo apt install nvidia-driver-525-server
```

安装完成后重启：

```
sudo reboot
```

```
nvidia-smi
```

```
Wed Jan  3 20:02:46 2024       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.147.05   Driver Version: 525.147.05   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:C3:00.0  On |                  N/A |
| 36%   49C    P0   110W / 350W |     50MiB / 24576MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A      1133      G   /usr/lib/xorg/Xorg                 36MiB |
|    0   N/A  N/A      1403      G   /usr/bin/gnome-shell               11MiB |
+-----------------------------------------------------------------------------+
```

## 安装cuda和cudnn

安装cuda

https://blog.csdn.net/qq_46107892/article/details/131453019

# 科学上网

删库跑路的clash for windows

https://dl.jichangzhu.com/clients/clash_for_windows/releases/latest/

手动配置系统代理端口为7890

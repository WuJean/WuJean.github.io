---
title: Esxi下配置ubuntu环境及tesla p4驱动安装
date: 2023-04-04 17:15:35
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

本文记录了在nuc9上的esxi环境下安装ubuntu22.04虚拟机，并安装tesla p4驱动的过程。

笔者的windows电脑嫌麻烦没装双系统，虽然wsl2用起来非常丝滑，但由于还没折腾好ssh连接wsl，以及win机子总不可能一直开着机，在外想用下linux环境时还是有一台本地的主机比较方便；有了上次过年背电脑回家的惨痛经历，以后回家就背nuc9回家了mac+nuc9+tesla p4，移动工作站yyds！

# esxi环境下安装ubuntu虚拟机

## 获取ubuntu ISO

在ubuntu官网下载iso文件，笔者暂时没有可视化桌面的需求，故下载LTS版本的ios

https://ubuntu.com/download/server

将下载好的镜像文件上传到esxi的目录中：

![image-20230405101938127](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230405101938127.png)

## 切换显卡直通

在 ESXi (VMware ESXi) 中，显卡直通（GPU Passthrough）是一种技术，允许将主机上的物理显卡直接分配给虚拟机，使虚拟机能够独占地访问显卡资源，而不与其他虚拟机或主机共享。这使得虚拟机可以在物理显卡上运行图形密集型应用程序，如游戏、图形渲染、人工智能等，而无需通过虚拟化软件进行图形加速或共享显卡资源。

暑假的时候迷上了捡计算卡垃圾；笔者的主力机上用的是tesla p4，nuc9小机箱当然得用上p40的儿子p4啦；不用独立供电，改完风扇后刚刚好可以塞进nuc9里，简直是天作之合；由于暂时没有搞vgpu的需求，故先将p4直通给ubuntu做日常炼丹使用。

![image-20230405102419330](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230405102419330.png)

## 配置虚拟机

选择cpu和内存选项，在CD/DVD驱动器中选择你上传的ubuntu镜像

添加PCI设备选择tesla p4；若没设置显卡直通，这一步将找不到PCI设备。

![image-20230405102917555](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230405102917555.png)

配置完后选择打开电源，若显示内存分配错误无法打开电源，重新编辑虚拟机，勾选预留所有客户机内容的选项：

![image-20230405103801593](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230405103801593.png)

接下来就是很正常的安装虚拟机的过程了，注意在安装完成后选择不进行更新立即重启，可以省点时间。

![image-20230405105631938](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230405105631938.png)

关闭电源后移除ISO文件，再次重启：

![image-20230405110446340](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230405110446340.png)

输入用户名和密码后登陆成功，完成系统的安装。

## 安装显卡驱动

！！！！！！！！我折腾了两天真的，整整两天，踩过了无数的坑，最后总结出一套快速安装的方法，我不知道里面哪些操作是必要的，但是这边建议完全按照我的操作来做！！！！！！！！

### 设置显卡直通

首先我们要确定你的显卡是否可以直通，如果可以的话在管理页面切换显卡直通：

![image-20230407151941111](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230407151941111.png)

最重要的一步来了！

在虚拟机设置中编辑以下几个选项：

取消勾选安全启动，这样在安装驱动的时候不会被限制而导致安装失败：

![image-20230407152514823](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230407152514823.png)

编辑高级配置：

- pciPassthru.64bitMMIOSizeGB = 你的显存*2
- pciPassthru.use64bitMMIO = TRUE
  - hypervisor.cpuid.v0 = FALSE	//让显卡以为自己不在虚拟环境中

然后重新引导你的esxi系统，这样才能在虚拟机中找到显卡

![image-20230407152629277](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230407152629277.png)

## 安装显卡驱动

### 禁用nouveau(nouveau是通用的驱动程序)（必须）

```
sudo vi /etc/modprobe.d/blacklist.conf
```

在最后两行添加：

```
blacklist nouveau
     
options nouveau modeset=0
```

在终端输入如下更新，更新结束后重启电脑（必须）

```
sudo update-initramfs –u
```

重启后在终端输入如下，没有任何输出表示屏蔽成功

```
lsmod | grep nouveau
```

### 下载驱动

这边有两种方法：

- 一种是ubuntu22.04支持apt下载驱动
- 另一种是自己到官网找安装包进行下载。

由于自动下载驱动的方式不适用与本环境，下载后会显示找不到设备，故只做记录

#### 自动下载驱动

添加第三方的驱动仓库：

```
sudo add-apt-repository ppa:graphics-drivers/ppa
```

更新软件列表:

```
sudo apt update
```

通过以下指令查看推荐的驱动版本:

```
ubuntu-drivers devices
```

apt自选驱动下载即可

#### 手动安装驱动

进入官网找到合适的驱动：https://www.nvidia.cn/Download/index.aspx?lang=cn

使用wget命令下载到虚拟机上并赋予执行的权限

```
sudo ./NVIDIA-Linux-x86_64-你的驱动版本.run -no-x-check -no-nouveau-check -no-opengl-files 
```

一路yes下来后重启查看是否安装成功

```
nvidia-smi 
```

![image-20230407154110529](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230407154110529.png)

## 总结

找资料的能力非常重要，一份好的资料能使得工作事半功倍

感谢大佬们写的文章，由衷感谢https://www.v2ex.com/t/905583提供的和我一样的环境的配置成功的案例，让我有了继续下去的动力。

接下来就是配置深度学习环境的过程了。

经过本地对驱动和虚拟机的折腾，让我对虚拟机的架构以及驱动的安装有了初步的认识，相信下一次配驱动一定能更加得心应手。有同样的问题的兄弟可以来dd我，一起交流学习解决问题！

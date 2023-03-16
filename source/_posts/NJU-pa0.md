---
title: NJU-pa0-配置环境
date: 2023-03-14 20:15:12
updated:
tags: NJU-pa
categories:
keywords:
description:
top_img:
comments:
cover:
---

# Installing GNU/Linux

手上一台主力windows台式机 一台mac 还有个ubuntu主机和腾讯云服务器

本pa最后要用GUI界面，为了避免折腾决定使用WSL2

## 安装WSL2

1. 打开命令行允许虚拟化：

```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

2. 打开主板BIOS找到允许虚拟化并勾选

3. 打开命令行输入：wsl --install自动下载wsl2

4. 点开windows应用商店搜索ubuntu，选择要求的版本并安装

![image-20230314202826308](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230314202826308.png)

# # First Exploration with GNU/Linux

打开terminal，开启新的世界

Many of you always use operating system with GUI, such as Windows. But the terminal is completely with CLI (Command Line Interface). Have you wondered if there is something that you can do it in CLI, but can not in GUI? Have no idea? If you are asked to count how many lines of code you have coded during the 程序设计基础 course, what will you do?

If you stick to Visual Studio, you will never understand why vim is called 编辑器之神. If you stick to Windows, you will never know what is 

Unix Philosophy

. If you stick to GUI, you can only do what it can; but in CLI, it can do what you want. One of the most important spirits of young people like you is to try new things to bade farewell to the past.

GUI wins when you do something requires high definition displaying, such as watching movies. 

Here

 is an article discussing the comparision between GUI and CLI.

按照要求使用几个命令：

```
wujean@DESKTOP-8K47HOB:~$ df -h   ## how much disk space Ubuntu occupies
Filesystem      Size  Used Avail Use% Mounted on
none            3.9G  4.0K  3.9G   1% /mnt/wsl
drivers         465G  294G  171G  64% /usr/lib/wsl/drivers
none            3.9G     0  3.9G   0% /usr/lib/wsl/lib
/dev/sdc       1007G 1007M  955G   1% /
none            3.9G   80K  3.9G   1% /mnt/wslg
rootfs          3.9G  1.9M  3.9G   1% /init
none            3.9G     0  3.9G   0% /run
none            3.9G     0  3.9G   0% /run/lock
none            3.9G     0  3.9G   0% /run/shm
none            3.9G     0  3.9G   0% /run/user
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
none            3.9G   76K  3.9G   1% /mnt/wslg/versions.txt
none            3.9G   76K  3.9G   1% /mnt/wslg/doc
drvfs           465G  294G  171G  64% /mnt/c
drvfs           3.7T  579G  3.1T  16% /mnt/d
```

可以看到我们WSL下的ubuntu可以访问我们机器上所有的磁盘 磁盘都挂载在/mnt下

/mnt/c 是系统盘c盘 /mnt/d 是一块4t的机械硬盘

```
wujean@DESKTOP-8K47HOB:~$ poweroff
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

实践证明WSL作为一个虚拟系统无法操纵系统关机

```
wujean@DESKTOP-8K47HOB:~$ sudo su
[sudo] password for wujean:
root@DESKTOP-8K47HOB:/home/wujean# exit
exit
```

获得了root权限 并使用exit退出

## Installing Tools

本节要求使用清华源镜像下载一些需要的软件 学习使用ubuntu下包管理软件apt

能翻墙的话直接设置一下终端的代理就可以无障碍下载软件了 不能翻墙的话可以自行寻找apt换源的教程

先ping下清华源的镜像检测下网络的连通性

```
wujean@DESKTOP-8K47HOB:~$ ping mirrors.tuna.tsinghua.edu.cn -c 4
PING bfdmirrors.s.tuna.tsinghua.edu.cn (202.204.128.61) 56(84) bytes of data.

--- bfdmirrors.s.tuna.tsinghua.edu.cn ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3097ms
```

居然ping不通 不过没关系继续下面的操作

```
bash -c 'echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse" > /etc/apt/sources.list'
```

将源的地址使用echo命令写入 /etc/apt/sources.list 中

注意命令前要加sudo升级为管理员权限

换源成功后执行一系列命令 先更新apt源 再尽情享受简单的软件安装的过程！

WSL记得在clash中开启允许局域网访问 再设置终端代理

```
sudo apt-get update
sudo apt-get install build-essential    # build-essential packages, include binary utilities, gcc, make, and so on
sudo sudo sudo sudo sudo sudo sudo apt-get install man                # on-line reference manual
sudo sudo sudo sudo sudo sudo apt-get install gcc-doc            # on-line reference manual for gcc
sudo sudo sudo sudo sudo apt-get install gdb                # GNU debugger
sudo sudo sudo sudo apt-get install git                # revision control system
sudo sudo sudo apt-get install libreadline-dev    # a library used later
sudo sudo apt-get install libsdl2-dev        # a library used later
sudo apt-get install llvm llvm-dev      # llvm project, which contains libraries used later
```

## Installing Chinese input method
WSL自带拼音的支持 保险起见为ubuntu安装中文语言包

```
sudo apt-get install language-pack-zh-hans         # 安装中文语言包
sudo apt-get install fonts-arphic-bkai00mp 
fonts-arphic-bsmi00lp fonts-arphic-gbsn00lp fonts-arphic-ukai     	# 安装字体
sudo dpkg-reconfigure locales				# 切换默认语言为中文
```

这步对新手真的特别友好 让笔者回忆起了为了在ubuntu上修改默认语言和输入法的悲惨回忆

## Configuring vim

没啥好说的，世界上最好的编译器vim

顺便推荐一个google上一个超好用的vim插件 

vimium https://chrome.google.com/webstore/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb

还有github上的一个入门项目 

Vim从入门到精通 https://github.com/wsdjeg/vim-galore-zh_cn

### Vim 哲学

Vim 采用模式编辑的理念，即它提供了多种模式，按键在不同的模式下作用不同。 你可以在普通模式 下浏览文件，在插入模式下插入文本， 在可视模式下选择行，在命令模式下执行命令等等。起初这听起来可能很复杂， 但是这有一个很大的优点：不需要通过同时按住多个键来完成操作， 大多数时候你只需要依次按下这些按键即可。越常用的操作，所需要的按键数量越少。

和模式编辑紧密相连的概念是 操作符 和 动作。操作符 指的是开始某个行为， 例如：修改、删除或者选择文本，之后你要用一个 动作 来指定需要操作的文本区域。 比如，要改变括号内的文本，需要执行 ci( （读做 change inner parentheses）； 删除整个段落的内容，需要执行 dap （读做：delete around paragraph）。

如果你能看见 Vim 老司机操作，你会发现他们使用 Vim 脚本语言就如同钢琴师弹钢琴一样。复杂的操作只需要几个按键就能完成。他们甚至不用刻意去想，因为这已经成为肌肉记忆了。这减少认识负荷并帮助人们专注于实际任务。

本节中还有一些很值得摘录的文段：

### 为什么要STFW?

你或许会想, 我问别人是为了节省我的时间.

但现在是互联网时代了, 在网上你能得到各种信息: 比如diff格式这种标准信息, 网上是100%能搜到的; 就包括你遇到的问题, 很大概率也是别人在网上求助过的. 如果对于一个你本来只需要在搜索引擎上输入几个关键字就能找到解决方案的问题, 你都没有付出如此微小的努力, 而是首先想着找人来帮你解决, 占用别人宝贵的时间, 你将是这个时代的失败者.

于是有了STFW (Search The F**king Web) 的说法, 它的意思是, 在向别人求助之前自己先尝试通过正确的方式使用搜索引擎独立寻找解决方案.

正确的STFW方式能够增加找到解决方案的概率包括：

- 使用Google搜索引擎搜索一般性问题

- 使用英文维基百科查阅概念

- 使用stack overflow问答网站搜索程序设计相关问题

- 如果你没有使用上述方式来STFW, 请不要抱怨找不到解决方案而开始向别人求助, 你应该想, "噢我刚才用的是百度, 接下来我应该试试Google". 关于使用Google, 在学校可以尝试设置IPv6, 或者设置"科学上网", 具体设置方式请STFW.

### 为什么不要用百度?

相信大家都用过百度来搜索一些非技术问题, 而且一般很容易找到答案. 但随着问题技术含量的提高, 百度的搜索结果会变得越来越不靠谱. 坚持使用百度搜索技术问题, 你将很有可能会碰到以下情况之一:

- 搜不到相关结果, 你感到挫败

- 搜到看似相关的结果, 但无法解决问题, 你在感到挫败之余, 也发现自己浪费了不少时间

- 你搜到了解决问题的方案, 但没有发现原因分析, 结果你不知道这个问题背后的细节

你可能会觉得"可以解决问题就行, 不需要了解问题背后的细节". 但对于一些问题(例如编程问题), 你了解这些细节就相当于学到了新的知识, 所以你应该去了解这些细节, 让自己懂得更多.

如果谷歌能以更高的概率提供可以解决问题的方案, 并且带有原因分析, 你应该没有理由使用百度来搜索技术问题. 如果你仍然坚持使用百度, 原因就只有一个: 你不想主动去成长.

After you are done, you should save your modification. Exit vim and open the .vimrc file again, you should see the syntax highlight feature is enabled.



### 为什么要这么麻烦?

搞了半天, 你发现其实也就是改动一个字符而已, 为什么不直接说清楚呢?

这是为了"入乡随俗": 我们希望你了解怎么用计算机思维精简准确地表达我们想做的事情. diff格式是一种描述文件改动的常用方式. 实际上, 计算机的世界里面有很多约定俗成的"规矩", 当你慢慢去接触去了解这些规矩的时候, 你就会在不知不觉中明白计算机世界是怎么运转的.

### More Exploration

[Linux入门教程](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/linux.html) [man快速入门](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/man.html) 必看！

作者说话真的是太好听了，忍不住一直摘录

#### 为什么要RTFM?

RTFM是STFW的长辈, 在互联网还不是很流行的年代, RTFM是解决问题的一种有效方法. 这是因为手册包含了查找对象的所有信息, 关于查找对象的一切问题都可以在手册中找到答案.

你或许会觉得翻阅手册太麻烦了, 所以可能会在百度上随便搜一篇博客来尝试寻找解决方案. 但是, 你需要明确以下几点:

- 你搜到的博客可能也是转载别人的, 有可能有坑

- 博主只是分享了他的经历, 有些说法也不一定准确
- 搜到了相关内容, 也不一定会有全面的描述

最重要的是, 当你尝试了上述方法而又无法解决问题的时候, 你需要明确"我刚才只是在尝试走捷径, 看来我需要试试RTFM了".

**Write a "Hello World" program under GNU/Linux**

```
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ vi main.c
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ g++ main.c
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ ls
a.out  main.c
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ ./a.out
Hello World
```

**Write a Makefile to compile the "Hello World" program**

[跟我一起来写makefile](https://seisman.github.io/how-to-write-makefile/index.html)

注意makefile的命令要以teb开头 否则会出现`Makefile:2: *** missing separator. Stop.`报错

```
hello: main.o
​        g++ -o hello main.o

main.o : main.c
​        g++ -c main.c
clean:
​        rm hello main.o -rf
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ vi Makefile
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ make
make: 'hello' is up to date.
wujean@DESKTOP-8K47HOB:~/NJU-pa/pa0$ ./hello
Hello World
```

**Learn to use GDB**

[A GDB Tutorial with Examples](https://www.cprogramming.com/gdb.html) 好长噢我明天再看

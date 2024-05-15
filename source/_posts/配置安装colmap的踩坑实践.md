---
title: 配置安装colmap的踩坑实践
date: 2024-05-10 13:15:34
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

配置的一个项目需要使用colmap，在编译安装colmap的过程中踩了很多坑，本文记录了安装colmap的全过程，以及如何解决报错的内容。

## 安装

### 获取源代码

项目要求使用colmap 3.8版本，在这出现了本次安装最大的一个坑：

colmap 3.8调用了cuda11中的一些函数，但是这些函数在cuda12中已经被废弃了，而colmap 3.9就可以适配cuda12。所以我们使用cuda11.8来编译

选择3.8版本

![image-20240510134224098](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20240510134224098.png)

```
git clone https://github.com/colmap/colmap
git checkout 43de802
```

### 编译

#### 安装依赖

来自默认 Ubuntu 存储库的依赖项：

```
sudo apt-get install \
    git \
    cmake \
    ninja-build \
    build-essential \
    libboost-program-options-dev \
    libboost-filesystem-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libeigen3-dev \
    libflann-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgtest-dev \
    libsqlite3-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libceres-dev
```

```
mkdir build
cd build
```

#### Cmake

~~官方给出的编译方式如下：~~

```
cmake .. -GNinja
ninja
sudo ninja install
```

这里有两个坑：

```
CMake Error at cmake/FindDependencies.cmake:125 (message):
You must set CMAKE_CUDA_ARCHITECTURES to e.g.  'native', 'all-major', '70',
etc.  More information at
https://cmake.org/cmake/help/latest/prop_tgt/CUDA_ARCHITECTURES.html
Call Stack (most recent call first):
CMakeLists.txt:85 (include)
```

需要我们手动设置 CMAKE_CUDA_ARCHITECTURES 以适应编译的cuda平台

但是我们选择native后虽然可以通过cmake，但是在编译的时候会出现另一个报错：

```
nvcc fatal : Unsupported gpu architecture 'compute_native' 
```

原因是无法识别GPU的计算能力。

正确的方式是（参考 https://github.com/colmap/colmap/issues/2464）：

```
检查 GPU 计算能力正在使用下面的命令
nvidia-smi --query-gpu=compute_cap --format=csv

然后在cmake中使用它：
cmake .. -GNinja -DCMAKE_CUDA_ARCHITECTURES=86
```

#### make

在编译快完成的时候会出现一个链接错误（如果你使用的是anaconda的话）

参考 COLMAP 安装报错 Undefined reference to libtiff4.0 解决办法

https://blog.csdn.net/weixin_49292745/article/details/123686007

发现在先前 cmake ..时，已出现warning：

```
runtime library [libgmp.so.10] in /usr/lib/x86_64-linux-gnu may be hidden by files in: /home/damon/anaconda2/lib
```

原来是用了anaconda的Qt，在colmap文件夹下的Cmakelist.txt中加入如下字段强制切换为系统Qt

```
SET(CMAKE_PREFIX_PATH "/usr/lib/x86_64-linux-gnu/cmake")
```

然后删除build文件夹中的内容再次重新进行编译，最终编译成功。

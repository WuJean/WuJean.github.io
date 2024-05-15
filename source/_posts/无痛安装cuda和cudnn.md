---
title: 无痛安装cuda和cudnn
date: 2024-05-10 12:53:11
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

介绍了一种无痛安装cuda和cudnn的方法

## 安装cuda

https://developer.nvidia.com/cuda-toolkit-archive?spm=5176.28103460.0.0.1d9d3da2AXMOWt

选择自己想安装的版本(本文以12.4为例)

![image-20240510125459110](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20240510125459110.png)

```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-4
```

```
sudo apt-get install -y cuda-drivers
```

添加环境变量：

```
echo 'export PATH="/usr/local/cuda-12.4/bin:$PATH"' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH="/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH"' >> ~/.bashrc
source ~/.bashrc
```

验证安装成功：

```
nvcc -V
```

## 安装cudnn

https://developer.nvidia.com/cudnn-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_network

同理：

```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cudnn
```

由于cudnn以库的形式安装，所以不用再添加环境变量

## 安装torch

https://pytorch.org/get-started/previous-versions/

选择适合自己版本的torch 由于cuda向下兼容，所以只需要关注torch1.x和troch2.x的区别

同样的，cuda也是向下兼容的



不过笔者在编译colmap的过程中报错:

```
/home/wujean/workspace/colmap/src/mvs/patch_match_cuda.cu(1629): error: identifier "cudaBindTextureToArray" is undefined
CudaSafeCall(cudaBindTextureToArray(poses_texture, poses_device_[0]->GetPtr()), "/home/wujean/workspace/colmap/src/mvs/patch_match_cuda.cu", 1629)
^

/home/wujean/workspace/colmap/src/mvs/patch_match_cuda.cu(1735): error: identifier "cudaUnbindTexture" is undefined
CudaSafeCall(cudaUnbindTexture(ref_image_texture), "/home/wujean/workspace/colmap/src/mvs/patch_match_cuda.cu", 1735);
^

/home/wujean/workspace/colmap/src/mvs/patch_match_cuda.cu(1736): error: identifier "cudaBindTextureToArray" is undefined
CudaSafeCall(cudaBindTextureToArray(ref_image_texture, ref_image_device_->GetPtr()), "/home/wujean/workspace/colmap/src/mvs/patch_match_cuda.cu"
```

这些错误提示表明在编译CUDA代码时遇到了未定义的标识符问题，具体是`cudaBindTextureToArray`和`cudaUnbindTexture`函数。这些函数属于较早版本的CUDA API，在某些较新的CUDA版本中已经被弃用或移除。从CUDA 11.0开始，纹理对象（texture objects）被推荐用来替代旧的纹理绑定方法。

所以cuda版本还是要慎重选择。

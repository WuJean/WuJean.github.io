---
title: Hexo博客配置多端编辑同步
date: 2023-03-15 18:01:07
updated:
tags: Blog
categories:
keywords:
description:
top_img:
comments:
cover: 
---

# 0x00 折腾的开始

上一篇博客讲了如何搭建一个hexo的博客，配置操作在任何一个平台都是通用的。与wordpress这类带管理页面的动态博客不同，hexo静态page是需要自己在配置好的机子上手动上传的，如果使用多设备的话就会很不方便。苦主在宿舍用的是windows台式机，出门一般带着mac笔记本，在外常常写不了博客，于是便决定配置一个多端编辑同步来治好我拖更的习惯！

# 0x01 多端同步原理

讲到多端同步我们就不得不提github了，我们的博客就是基于github page搭建的。一个很简单的思路就是在将我们的配置文件在每台机子上都配置一遍，然后在编辑结束后，未改动的机子将改动的内容同步，这样就能很简单的实现多端同步。

但是单纯的复制粘贴会面临每台机子独特的.git文件不同、本地分支冲突、ssh key冲突等问题；git的分支功能很好的解决了该问题。

`hexo g`将我们的源文件部署， `hexo d`上传的只是网页部署文件，这些文件上传到了 github的 `master`分支，我们在另一台电脑上如果能够拥有源文件的话，同样将这些部署文件上传到 github 的 `master`分支即可，那么其实我们要做的就是备份源文件。

那么我们可以在github的blog仓库**新建一个分支，存储源文件，亦或者新建一个仓库，存储源文件**即可，这样我们就可以在多个终端间同步源文件，而后就可以进行博客文件的终端同步了。

# 0x02 具体操作

在你的username.github.io仓库中新建分支存储源文件的hexo分支，并设置为默认分支：

![image-20230316200020542](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230316200020542.png)

## 本地电脑操作

本地电脑任意目录下clone原仓库的内容，本步是为了获取仓库新分支的.git信息，clone结束后将文件夹中除了.git文件外其他文件删除：

```bash
git clone git@github.com:<your rep url ,eg :name.github.io.git>
```

将源文件（blog）文件夹中的除了.git文件全部复制进新文件夹中，如果你的主题是用clone来的，记得删除themes主题文件夹的.git文件，不然可以会因为git冲突无法实现上传。

上传文件到hexo分支

```text
git add .
git commit -m "backup blog source file"
git push 
```

如果没有报错，此时github端应该就可以看到备份的源文件了。

## 远程电脑操作

从仓库中将备份的源文件clone下来，在新文件夹中按照配置博客的方式安装好hexo博客的框架

首先进行一些基础配置，安装git nodejs 配置git连接Github

```text
npm install hexo-cli -g			# intall hexo

# 在该电脑的本地文件夹下clone Blog源文件
git clone <url>
```

clone 成功后，进入blog文件夹下，安装之前安装的插件

```text
npm install
npm install hexo-deployer-git --save
```

然后就可以在新电脑上写博客了，将博客部署到网站后，记得备份源文件

```text
git add .
git commit -m ""
git push 
```

## what‘s more

在远程电脑git push的过程中，github要求提供账号和密码的信息，但是苦主输入正确的账号密码后提示鉴权失败，几经查阅后发现github已经取消了使用账户名和密码登录的方式。我们需要获取token来获取对仓库的访问权限。

![image-20230316201108111](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230316201108111.png)

记得勾选仓库的权限：

![image-20230316201140232](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230316201140232.png)

随后将获取的key输入到密码中就可push成功！key只出现一次要注意保存。

# 0x03 如何使用

在每次写博客之前先

```
git pull 		//拉取最新的源文件
```

写完后

```
git push		//备份源文件
```

现在开始你快乐的写博客之旅啦，一定记得不能再鸽噢！


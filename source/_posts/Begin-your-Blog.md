---
title: Begin your Blog
date: 2023-03-04 09:34:04
tags: Blog
---



# 0x00 为什么要写博客

学了很多东西，中断一段时间的学习后总觉得啥都忘掉了；折腾了很多东西，但再次折腾的时候总是要重新找教程；踩过很多坑，但是再次遇到的时候总要重新踩一般；做过什么事情，别人问的时候还得翻历史记录找链接。

如果你有遇到过以上的这些问题，那么你就该开始写博客了。诚然在写博客中会遇到很多的问题，在看完别人很好的技术文章后想自己总结一遍，但总觉得是在照抄别人的东西；自己搭建博客的时候会遇到许多问题，耗费很多的时间和精力；总是坚持不下去写博客，三分钟热度。但是这个折腾和总结的过程正是我们搭建自己的知识框架的过程，看过的文章再好，动动小手copy的时候也会发现很多忽略的细节，完成这样一个写的过程往往更能体会到作者的深意，缕清行文的思路；折腾得越久，投入得 时间越多，也会更想把自己得博客经营好；当别人问你问题的时候甩上自己文章得链接，岂不美哉！

# 0x01 选择博客框架

我从大一接触到云服务开始，使用了很多种博客框架，其中最方便得就是云服务器上基于宝塔面板的Wordpress框架，使用方便而且容易配置域名放在公网访问，但是要进行备案以及内容审查，面临着域名、服务器过期等问题。

后来又尝试了Hexo+githubpage的框架，体验非常好，无需备案，高度的可定制性，网上教程资源多，但是对刚刚入门的同学操作会有点难度。

最近在新买的homelab上搭建了内网的基于ubuntu docker的wordpress，配置了frp转发访问，折腾完觉得索然无味，于是便决定回归Hexo。折腾那么久后才觉得博客的本质在于文章而不在于框架，框架简单易用、能为写文章提供最大的便利即可。

本文记录了基于github page搭建Hexo博客，通过PicGo+github搭建图床，使用Typora+PicGo编辑文章 的工作流的流程。发布文章便捷，博客访问方便，且不用但心审查和过期。

# 0x02 搭建Hexo博客

## 2.1 准备工作

- github账号

在github中新建仓库，仓库名为yourusername.github.io,在setting中找到github page并配置publish

![image-20230304101702336](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230304101702336.png)

- 安装好Git

- 安装好NodeJs

![image-20230304101837180](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230304101837180.png)

### add ssh key

可以使用 SSH（安全外壳协议）访问和写入 上的存储库中的数据。 通过 SSH 进行连接时，使用本地计算机上的私钥文件进行身份验证。 有关详细信息，请参阅“[关于 SSH](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/about-ssh)”。

生成 SSH 密钥对后，必须将公钥添加到 以启用帐户的 SSH 访问。

打开命令行并输入：

```
ssh-keygen -t rsa -C 'your_email@example.com'
```

生成你的密钥。

![img](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1AHCozuJqdGsaIacfQwK%2Fuploads%2F6r9XErDPUeAR03LTJtWO%2Fimage.png?alt=media&token=03ee0429-77a5-476a-a676-ee4274b78453)

可以看到你的密钥保存在/home/your_name/.ssh下，打开该文件夹并复制刚刚生成id_rsa.pub中的内容

随后打开你的github setting在ssh keys处添加你的公钥，完成添加。

![img](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1AHCozuJqdGsaIacfQwK%2Fuploads%2FBjdJIPIbcrBDFfStreOd%2Fimage.png?alt=media&token=2b6984bf-6923-415e-ad82-713f45268f37)

最后建立一下ssh连接确认是否成功。输入：

```
ssh -T git@github.com
```

![img](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1AHCozuJqdGsaIacfQwK%2Fuploads%2FtkkA5oCVxx3aDmTJbQzP%2Fimage.png?alt=media&token=3900ee84-7f77-4a5a-87af-094a06dfd874)

## 2.2 安装Hexo

我们采用`Hexo`来创建我们的博客网站，`Hexo` 是一个基于`NodeJS`的静态博客网站生成器，使用`Hexo`不需开发，只要进行一些必要的配置即可生成一个个性化的博客网站，非常方便。点击进入 [官网](https://hexo.io/zh-cn/)。

安装 Hexo

```
npm install -g hexo-cli
```

查看版本

```
hexo -v
```

创建一个项目 hexo-blog 并初始化

```
hexo init hexo-blog
cd hexo-blog
npm install
```

本地启动

```
hexo 
hexo server
```

启动后的页面如图所示，我们可以在本地预览博客

![image-20230304102134819](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230304102134819.png)

## 2.3 将博客推送到github page

安装hexo-deployer-git

```
npm install hexo-deployer-git --save
```

修改**根目录**下的 _config.yml，配置 GitHub 相关信息

```
deploy:
  type: git
  repo: your repo address
  branch: main
  token: your token
```

![请添加图片描述](https://img-blog.csdnimg.cn/2350558a10d94c8ab4959b04771d2bcc.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhb3JvbmdrZQ==,size_16,color_FFFFFF,t_70)

执行生成博客并推送到远程的命令

```
hexo g -d
```

浏览器访问 https://yourunsername.github.io, 部署成功

![image-20230304102636388](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230304102636388.png)

## 2.4 Hexo 命令

```
hexo n "标题"			//生成文章并存储到source/_posts
hexo g 				//生成博客页面
hexo s 				//本地预览文章
hexo d				//推送文章到github page
```

使用简单的命令就可以完成博客的生成和部署，注意在未添加ssh key时使用hexo d会出现权限不够无法推送的问题，请务必为本机添加ssh key。

## 2.5 修改Hexo主题

使用Next主题

将主题的源码放入blog文件夹中的themes文件夹

![image-20230304103846611](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230304103846611.png)

修改blog文件夹中的_config.yml文件,将主题修改成对应的主题名称

![image-20230304104244994](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230304104244994.png)

重新生成并部署博客

## 2.6 小结

至此博客的部署全部完成，读者可以自己搜索相关配置文章，分别修改博客目录和主题目录下的_config.yml文件，注意要区分博客目录和主题目录！添加插件各种魔改来完善自己的博客。反正我是不折腾了。

# 0x03 PicGo搭建图床

博客文章的图片需要找个地方存放，全部存放在本地占用过大，引用别人的图片说不定哪一天就挂了；于是决定白嫖github的存储空间，毕竟能访问github page的也应该得能访问github

## 3.1 配置github

创建一个新的公共仓库，并获取github的access tokens

![img](https://pic2.zhimg.com/80/v2-346da4ccf189eb5997abe2fadadca331_720w.webp)

将获取的token保存起来

## 3.2 配置PicGo

https://github.com/Molunerfinn/PicG 下载PicGo并打开

![image-20230309093148974](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230309093148974.png)

- 输入你设定的仓库名（注意斜杆前后无空格）

- 将分支修改为main
- 将上一步得到的token输入进去

若未开启代理则有可能无法访问github，故我们在设置里面设置一下代理

![image-20230309093340529](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230309093340529.png)

随意上传一张图片，上传成功则配置成功！

# 0x04 配置Typora+PicGo工作流

下载Typora 支持正版！

Typora有个很好的地方就是可以与PicGo无缝衔接，在文档中插入图片可以直接上传到图床中，使得工作流变得十分流程。

![image-20230311092946660](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230311092946660.png)

按照下图设置便可完成配置。

# 结语

现在就可以开启你的博客之旅啦！记得保持更新，千万不要鸽！更重要的是注重内容的本身，注重这样一个积累的过程。

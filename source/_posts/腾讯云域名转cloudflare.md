---
title: 腾讯云域名转cloudflare
date: 2023-03-26 15:22:46
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

折腾了个homelab，已经弄好了frp内网穿透，想搭建一些自己的服务放到公网上访问；买好域名做好解析和ssl之后短暂的访问成功了一会，再次访问便被要求备案，就像下面这样：

![image-20230326224450410](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230326224450410.png)

原因是当域名解析到国内的服务器时都需要备案处理。

虽然我没有搞奇奇怪怪东西的想法，我的服务也没有提供给无法科学上网的人的必要，一大堆审核以及不定时的电话盘问很坏人的心情，我搭个网站搞搞服务 整得像搞破坏一样。为了更加自由的域名解析以及为将来更换国外的vps做准备，我决定将域名解析迁移到知名的cdn服务商cloudflare上。

# 折腾的过程

## 注册cloudflare并添加域名

**官网：**[Cloudflare - The Web Performance & Security Company](https://link.zhihu.com/?target=https%3A//www.cloudflare.com/)

首先打开 Cloudflare 的官网，点击 **Sign Up** 注册账号。输入邮箱和密码，点击 Create Account 即可。

![image-20230326224930360](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230326224930360.png)

注册完成后点击Add a Site添加自己的域名

![image-20230326225032073](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230326225032073.png)

选择免费的服务即可

## 更换dns服务器

![image-20230326225358962](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230326225358962.png)

由于我们的域名是在腾讯云购买的，我们需要手动指定该域名的dns服务器

打开腾讯云的域名界面：

![image-20230326225657464](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230326225657464.png)

用cloudflare的两个域名替代腾讯云默认的dns服务器，等待五分钟左右我们就能看到域名的dns状态变为其他，现在我们只需要在cloudflare上配置域名解析的规则。

## 将域名与github page关联



我们一般设置两条分别指向服务器 IP 地址的 **A 记录**即可，示例如下：

- **第一条：**Name：`@`，IPv4 address：`服务器 IP`
- **第二条：**Name：`www`，IPv4 address：`服务器 IP`

这样就把 `example.com` 和 `www.example.com` 两个域名都指向服务器了，待解析生效后就可以在网络上访问到了。

我们只需要将我们的域名指向github的四个IP地址中至少一个：

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

接着在你的github.io仓库中设置CNAME映射

### CHAME

在GitHub Pages中，CNAME记录是用于将自定义域名绑定到您的GitHub Pages网站的记录类型。通过将CNAME记录添加到您的DNS配置中，您可以将自定义域名指向您的GitHub Pages网站，从而允许访问者通过您的自定义域名访问您的网站，而不是使用默认的github.io域名。

具体来说，CNAME记录告诉DNS服务器将您的自定义域名指向GitHub Pages的服务器。一旦CNAME记录生效，GitHub Pages将自动识别您的自定义域名，并将其用作访问您的网站的标准网址。




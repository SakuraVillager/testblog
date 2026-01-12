---
title: 利用Cloudflare R2 + PicGo搭建低成本图床
published: 2025-08-29
description: ""
image: ""
tags: []
category: ""
draft: false
lang: ""
---
> 前提条件：一个Cloudflare账号、一个域名

没错，就这么简单。没有Cloudflare账号的可以访问Cloudflare注册账号 [Cloudflare Dashboard | Manage Your Account](https://dash.cloudflare.com/sign-up)，域名可以使用二级域名。如果使用免费的二级域名就是0成本了。
## Cloudflare端配置
点击`R2对象储存`，添加一个付款方式并点击`将R2订阅添加到我的账户`。
图中A类操作代表每月最多的写操作，B类操作代表每月最多的读操作。
![](https://img.57314322.xyz/img/20250829130030138.png)
> 从图中我们可以看到，虽然添加了一个付款方式，但只有当超出限额了才会进行收费，而且收费相较于国内一些大厂来说已经很便宜了。他为用户提供了10GB/月的免费空间，读写操作次数也很多，作为个人图床完全够用。
 
 随后点击`创建储存桶`，存储桶名称随便写一个，地区选择服务器所在地区即可，默认存储类选择标准，然后点击创建。
看到这个界面，就说明创建成功了。
![](https://img.57314322.xyz/img/20250829130106403.png)
但是现在这个图床还没有公网链接，我们依次点击`设置`-`公共开发URL`-`启用`，输入allow并确认，这样图床就可以被访问到了。
![](https://img.57314322.xyz/img/20250829130127958.png)
现在我们上传一张图片测试一下，点击上传的图片名称就可以到详情页。
![](https://img.57314322.xyz/img/20250829130416597.png)
可以看到他已经给我们的图片生成了url，并且可以在浏览器中访问到[上传的示例图片](https://pub-0955397f45d44eeaa359d757bfbb6542.r2.dev/infinity-13582.jpg)
## 使用自己的域名
如果发现过了一会链接访问不了了，就需要这个方法。
首先将自己的域名托管到Cloudflare。
在账户主页中点击`加入域`，跟随引导操作选择免费套餐注册即可。（后续我这边没有新的域名不方便演示）
![](https://img.57314322.xyz/img/20250829130824053.png)![](https://img.57314322.xyz/img/20250829130509520.png)
托管到Cloudflare后，回到`R2对象储存`，点击`设置`-`自定义域`-`添加`，输入自己的域名，之后就能浏览器中通过`https://<自定义域名>/<文件名>`来访问存储桶里的文件了。这时候可以把公共url关掉。
![](https://img.57314322.xyz/img/20250829130909576.png)
自此，Cloudflare端的配置就完成了，但是每次写博客都需要上cloudflare上传图片非常繁琐，于是我们需要用一个软件来辅助我们更方便的上传与管理图床。
## 结合PicGo优雅食用
>项目GitHub地址：[Molunerfinn/PicGo: :rocket:A simple & beautiful tool for pictures uploading built by vue-cli-electron-builder](https://github.com/Molunerfinn/PicGo)

使用文档：[PicGo is Here | PicGo](https://picgo.github.io/PicGo-Doc/zh/guide/#picgo-is-here) 
官网对PicGo的描述是“一个用于快速上传图片并获取图片URL链接的工具”，它提供了便捷的图片上传与管理方法。[下载]( https://github.com/Molunerfinn/PicGo/releases)并安装PicGo。
PicGo本身提供了很多平台的图床例如GitHub、又拍云、阿里云oss等，~~本人就是从GitHub转来的~~，但是没有提供Cloudflare的接口，需要一个插件`s3`来帮助我们实现。点击`插件设置`，搜索`s3`并安装，**随后重启应用**。
![](https://img.57314322.xyz/img/20250829131012602.png)
重启后，在`图床设置`下找到`Amazon S3`，新建设置。
![](https://img.57314322.xyz/img/20250829131035532.png)
![](https://img.57314322.xyz/img/20250829132902114.png)
|    设置名称     |                                   值                                   |
| :---------: | :-------------------------------------------------------------------: |
|   图床配置名称    |                                  随便填                                  |
| 应用秘钥ID和应用秘钥 |                                 后文会讲                                  |
|     桶名      |                      刚刚新建桶的名字，例如我的就是`blogimage`                       |
|   上传文件路径    | 上传到R2中的文件路径，我使用`img/{fileName}.{extName}`，功能是上传到R2的img文件夹下，并保留文件名和扩展名 |
|     地区      |                               填入`auto`                                |
|    自定义节点    |                                 后文会讲                                  |
|    自定义域名    |  访问你图片的url模板。如果不是采用自定义域名的请忽略。我的模板https://域名/img/{fileName}.{extName}  |
### 应用秘钥ID和应用秘钥
在R2储存的`概述`界面中点击`API`-`管理API令牌`，随后点击`创建 Account API 令牌`，令牌名称随便取一个，权限选择对象读和写，指定储存桶为我们新建的那个桶，点击新建令牌。这样我们就得到了api令牌。其中`访问密钥ID`就是`应用秘钥ID`，`机密访问密钥`就是`应用秘钥`。
![](https://img.57314322.xyz/img/20250829133012368.png)
![](https://img.57314322.xyz/img/20250829133106268.png)
### 自定义节点
在R2储存的`概述`界面中点击`API`-`将R2与API配合使用`。将弹出的窗口下方的网址填写入`自定义节点`。
![](https://img.57314322.xyz/img/20250829133118766.png)
![](https://img.57314322.xyz/img/20250829133129948.png)
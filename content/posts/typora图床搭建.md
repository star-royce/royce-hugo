---
title: "Typora图床搭建"
date: 2020-07-28
slug: "typora upload picture"
draft: false
tags:
- Tech
- BasicTool
categories:
-
---


# 腾讯云COS + PicGo + Typora 上传图片

## 下载PicGo

https://github.com/Molunerfinn/picgo/releases

## 腾讯云COS

腾讯云的COS是一种对象存储服务，支持不同格式的文件上传以及下载等访问。存储容量，访问以及CDN分开收费。目前有按量收费以及按资源包收费，如果是自用（比如博客或者存储照片等），选择按量收费是比较好的选择。

按照一下方式操作：

![image-20200705120457509](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120457.png)

![在这里插入图片描述](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120525.png)

![在这里插入图片描述](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120539.png)

![在这里插入图片描述](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120553.png)

## 配置Typora和PicGo

![在这里插入图片描述](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120630.png)

配置信息来自于刚才在腾讯云的配置

![image-20200705120821976](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120822.png)

最后设置完成我们验证一下

![在这里插入图片描述](https://pic-1302575189.cos.ap-guangzhou.myqcloud.com/mcr/img20200705120911.png)


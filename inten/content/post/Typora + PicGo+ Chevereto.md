---
title: "Typora + PicGo+ Chevereto踩坑指南"
date: 2021-03-13T15:07:34+08:00
tags: ["工具"]
ShowToc: true
TocOpen: true
draft: false
---

### 目的：

解决md文件图片传输不方便的问题，将图片上传到图床，方便分享md文件。

### 运行环境

win10

version 0. 9. 98 (beta)★website★@typora

picgo-2.2.2

### 1.去官网下载最新的typora和picgo，并安装

### 2.配置typora

![image-20210310144511055](https://www.kro1lsec.com:442/images/2021/03/10/20210310144511.png)

### 2.配置picgo

修改端口

![image-20210310143246950](https://www.kro1lsec.com:442/images/2021/03/10/20210310143247.png)

安装插件

![image-20210310143133724](https://www.kro1lsec.com:442/images/2021/03/10/20210310143133.png)

配置Chevereto

注意仔细核对url

```js
"url": "https://个人服务器Cheverto图床域名或地址/api/1/upload"
```

![image-20210310143416160](https://www.kro1lsec.com:442/images/2021/03/10/20210310143416.png)

记得**打开时间戳重命名**，Chevereto不允许上传同名图片，不然会出现以下错误。

![image-20210310143609783](https://www.kro1lsec.com:442/images/2021/03/10/20210310143609.png)

我测试时

picgo的时间戳不会重命名typora的验证图片时的测试图片，所以会出现第一次验证成功，之后失败的情况。

实际功能正常。

### 踩坑与调试

**1.为什么原来的URL不行**

可以说得按照官方文档来，乱填url当然不行，感觉这样的回答太牵强。

我推测是图像界面面向用户上传图片的调用方法 和 为PicGo这类用于快速上传图片并获取图片 URL 链接的工具的接口不一样。后者至少比前者多一个获取图片 URL 链接的返回数据。如果用前者的url那typora就收不到图片的链接，功能会无法实现

**2.为什么现在这个URL可以**

按照chevereto官方文档的说明（https://v3-docs.chevereto.com/API/V1.html#api-call）

我们最好采用“POST”的形式来传数据。默认的上传url为：
`https://mysite.com/api/1/upload`

**为什么锁定是url的问题？**

假设Typora + PicGo+ Chevereto这三个软件及插件都没问题，key也没问题。

PicGo的本质还是收发数据包吧，我没有去看源码，但感觉和web端的通信是通过给url传不同的参数来实现。

要么传的参数有问题，要么url有问题，逐个排查也可。参数大概是软件设置的选项，然后我也一直在调，常见的问题都搜过了，小概率是问题太偏。但url真的很可疑呀，我填的时候没有想太多，就理所当然地复制了。连文档都没查，参考的博客看得也不仔细。
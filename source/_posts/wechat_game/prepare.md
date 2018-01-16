---
title: 微信小程序开发工具 linux 运行
date: 2018-01-14
categories: wechat
tags: [wechat_game]
---

首先，微信小程序开发工具是用nw.js实现的，[nw.js](https://nwjs.io/)(直接从DOM中调用所有的Node.js模块)
本身是跨平台的，但是微信只出了开发工具的windows和mac版。
下面介绍 linux 系统如何运行开发工具， 以 ubuntu 为例。
<!-- more -->

### 1.nw 
下载nwjs sdk（需要devtool的支援） 压缩包之后解压;

### 2.设置 path 变量；
```
$ vim ~/.bashrc # 视个人情况选择文件
# 加入步骤一中解压目录， 
# 例如： export PATH="$PATH:/home/kk/Downloads/nwjs-sdk-v0.27.4-linux-x64"
```

### 3.package.nw 
在 windows 机器安装开发工具， 找到安装目录，将 package.nw 文件夹拷入 ubuntu 系统；
### 4.运行
```
$ cd package.nw
$ nw .
```
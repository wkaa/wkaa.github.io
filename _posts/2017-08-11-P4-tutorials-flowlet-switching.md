---
layout: post
title: P4入门（二）--flowlet switching（simple router）
categories: P4
description: P4教程第二弹！这次自信一点了！
keywords: P4 SDN Switch Route
---

　　P4教程第二弹！感觉入门些了，通过看文档还有查资料，大部分内容能够理解了，这里就介绍一下tutorials的第二个练习，Let's start！

　　首先，flowlet switching是干什么的，大致就是做一个ecmp流控的，ecmp是什么？google一下，介绍和例程戳这里[tutorials](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015#obtaining-required-software)

## 源码

　　源码太多了，不好一口气贴，我讲到哪里贴到哪里吧，源码连接在[这里](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015/flowlet_switching)

## 分析

　　这次需要结合p4源码与commands.txt一起来看，commands.txt定义了表与初始表项，需要在查表时用到，接下来从头部定义开始。

### 头部定义

　　headers.p4中定义了用到的头部，然后在主文件中`#include`，其中定义了以太网头部，ipv4头部，tcp头部。

　　intrinsic.p4中定义了一个元数据类型的头部，这种按我理解就算变量了吧。

　　其他其他还有一些定义就先不一一说了。

### 解析

　　依旧从start开始，一开始就先跳到以太网帧头部解析，提取了头部，判定是ipv4后跳到ip解析，在ip解析这里提取了ip头部后，判断协议类型，判定是tcp后跳到tcp解析，这里只提取一下tcp头部，然后直接转到ingress。

### 表匹配

　　在ingress中，一共有4个或5个表要匹配，

　　未完待续

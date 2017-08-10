---
layout: post
title: P4入门（一）--Source Routing
categories: P4
description: 对P4一脸懵逼，一点点总结分析吧
keywords: P4 SDN Switch Route
---

　　第一次听说P4这东西，目前还是一脸懵逼状态，但是一点点记录下来可能会好理解一些。

## P4是什么

　　P4(Programming protocol-independent packet processors)是由Pat Bosshart等人提出来的高级“协议独立数据包处理编程语言”，如OpenFlow一样是一种南向协议，但是其范围要比OpenFlow要大。不仅可以指导数据流进行转发，还可以对交换机等转发设备的数据处理流程进行软件编程定义，是真正意义上的完全SDN。

　　按我目前理解就是针对于交换机等转发设备的一种编程语言实现转发流程吧

## Source Routing

　　鉴于我现在什么都还不懂，直接先走一个实验亲身体会一下吧，在P4的github上有一个[tutorials](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015#exercise-1-source-routing。)里面有Source Routing的详细介绍，设计了一个三台交换机与两台host的拓扑

　　介绍不下去了，还是先学学再来看吧


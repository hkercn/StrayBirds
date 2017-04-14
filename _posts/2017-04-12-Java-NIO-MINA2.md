---
layout: post
title: 视频直播项目总结之PC直播
category: 技术
comments: false
---

<br/>

## 前言 ##

“我们必须全力奔跑,才能停留在原地”

## PC直播流程概述 ##

项目同后台服务器的网络通信框架基于MINA2.0.7搭建，PC直播模块的音视频模块也是依赖后台C++业务模块通过MINA2框架将音视频数据推送至客户端。

关于MINA2在该项目中的应用总结，请参考

首先，从App启动加载so文件-->进入直播间-->播放音视频-->退出直播间，播放结束。简要的流程图如下

![pcstream.png](https://hkercn.github.io/blog/images/pcstream.png)

由于视频流数据的解码、界面绘制同音频数据的解码播放这两个业务流程是独立进行的，因此我们这里也分开分别进行讨论总结。

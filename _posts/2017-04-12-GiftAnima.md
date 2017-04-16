---
layout: post
title: 视频直播项目总结之送礼动画
category: 技术
comments: false
---


## 基础知识 ##

![Android 属性动画 源码解析 深入了解其内部实现](http://blog.csdn.net/lmj623565791/article/details/42056859)

![Android 属性动画（Property Animation） 完全解析 （上） ](http://blog.csdn.net/lmj623565791/article/details/38067475)

![ Android 属性动画（Property Animation） 完全解析 （下） ](http://blog.csdn.net/lmj623565791/article/details/38092093)

<<Java并发编程:核心方法与框架 (高洪岩) >>

## 动画需求的分析和实现 ##

### 1. 如何实现UI给出的动画效果? ###

  * 简单动画，使用View Animation实现即可。
  
  * 对于稍微复杂些的动画效果，可以将动画效果进行细分，如可拆分成X/Y轴位置变换动画、透明度/旋转动画/缩放等动画，然后通过组合的方式实现复杂的动画需求。
  
### 2. 如何以队列形式有序展示大量密集的动画? ###

  * 客户端通过接受服务器的送礼通知，来判断房间内某个用户发送的礼物类型、名称、价格、图标等信息的。客户端也是在接收到这些通知数据的时候，进行的送礼动画展示。
  
  * 那么从性能和体验的角度来考虑，送礼动画的队列处理操作应放在客户端处理比较适宜。
  
### 3. 客户端实现大量密集型送礼动画的队列展示处理   ###
  
  构建一个动画展示任务队列的单例工具类RevGifQueue，内部定义一个接收到的送礼通知消息队列缓冲区。
  
  ```java
  
  LinkedBlockingQueue<UserGiveGiftNotify> giftNotifyQueue;
  
  ```
  
  

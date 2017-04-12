---
layout: post
title: 视频直播项目总结之直播核心技术相关概念、第三方平台及开源库的选择
category: 技术
comments: false
---

<br/>
## 基础知识

对于视频直播相关的基础知识点，其实是可以从一些国内视频云解决方案平台的帮助文档或者相关博文中学到不少东西的。这里列出部分引用做参考:

- [直播基础概念-来自腾讯云直播平台的帮助文档](https://www.qcloud.com/document/product/454/7937)

- 来自UCloud的直播技术细节博文分享

   * [关于直播，所有的技术细节都在这里了（一）](http://blog.ucloud.cn/archives/694).

   * [关于直播，所有的技术细节都在这里了（二）](http://blog.ucloud.cn/archives/699).

   * [关于直播，所有的技术细节都在这里了（三）](http://blog.ucloud.cn/archives/760).

   * [关于直播，所有的技术细节都在这里了（四）](http://blog.ucloud.cn/archives/796).

- 来自知乎视频直播相关的话题

   * [哪些地方可以学习视频直播技术？](https://www.zhihu.com/question/23651189) 

   * [如何搭建一个完整的视频直播系统？](https://www.zhihu.com/question/42162310) 

   * [最近市面上很火爆的17、花椒、虎牙直播、periscope的直播功能，是自研还是第三方直播SDK服务？](https://www.zhihu.com/question/36076688/answer/101142263) 

- [如何开发出一款仿映客直播APP项目实践篇](http://www.jianshu.com/p/b2674fc2ac35#)
<br/>

## 国内外视频云解决方案的选择注意点

####1. App的用户市场

  也即目标收众用户所在的国家区域。

  如果是国内，那么目前常见的视频云平台都是可以满足的，因为CDN等的因素影响还没那么大；

  但如果是国外，那么就要综合考虑目标视频云平台在国外服务器节点是否满足需要了，是否可以开放对接国外CDN，特定国家用户推拉流时是否链路的他国的服务器节点等等。

####2. 平台的文档和客服

  需要详细考量平台提供的文档是否细致全面，演示demo所覆盖的产品功能是否全面；平台技术支持对问题的跟进效率；

####3. 产品质量考量

  这一点是经过上述两点调研后，需要技术上深入一点考量的地方。
  比如经过如上两点考量后我司的App最终排除了阿里/百度的视频云解决方案，而采取了腾讯云，但是经过一段时间的开发迭代后发现，国外部分国家用户经常会反映直播卡顿的问题。幸好TXSDK推拉流端对基本的网络状态/推拉流质量反馈机制做的还比较到位，结合推拉流端TXSDK的反馈机制，技术上增加了推拉流质量判断上报服务器的逻辑，最终排查出部分卡顿现象是出于部分主播于户外网络条件欠佳的环境下导致的推流端卡顿延迟，从而导致了全局拉流端卡顿延迟的问题。

## 第三方的开源库调研

如若有较强研发能力的团队，那么可以考虑自行实现一套可用可定制的解决方案，否则还是参考下开源的解决方案吧，毕竟时间不会等人。在调研考量的时候，也是需要结合自身产品的用户群分步、服务器现阶段的技术实现状态，详细测试来得出比较靠谱的方案的。

#### 拉流方案

* [ijkplayer](https://github.com/Bilibili/ijkplayer)  推荐

  Android/iOS video player based on FFmpeg n3.2, with MediaCodec, VideoToolbox support. 
  platform: API 9~23
  hw-decoder: MediaCodec (API 16+, Android 4.1+)
  cpu: ARMv7a, ARM64v8a, x86 (ARMv5 is not tested on real devices)

* [PLDroidPlayer](https://github.com/Bilibili/ijkplayer) 推荐

  基于 ijkplayer ( based on ffplay )
  最低支持Android API 9
  支持 RTMP 和 HLS 协议的直播流媒体播放

* [sinavideo_playersdk](https://github.com/SinaVDDeveloper/sinavideo_playersdk) 推荐

  最低支持Android API 9
  内核已经开源

* [ffplayer](https://github.com/FFmpeg/FFmpeg/blob/master/ffplay.c)

  开发成本较高，需要一定的JNI/C语言的开发能力。
 
* [ExoPlayer-with-RTMP-and-FLV-seek](https://github.com/ant-streaming/ExoPlayer-with-RTMP-and-FLV-seek)

* [JieCaoVideoPlayer](https://github.com/lipangit/JieCaoVideoPlayer)

   Support https and rtsp
  
  
#### 推流方案

* [yasea](https://github.com/begeekmyfriend/yasea)  推荐

  最低支持android API 16.
  H.264/AAC hard encoding./H.264 soft encoding.
  
* [librestreaming](https://github.com/lakeinchina/librestreaming)

  Support Android 4.3 and higher (Android 6.0/7.0-preview tested)
  
  Filters support soft mode (CPU processing) and hard mode (GPU/OpenGLES rendering)[支持两种滤镜模式：使用cpu的软模式和使用gpu(opengles)的硬模式]

* [OpenLiveAndroidPusher](https://github.com/devillee/OpenLiveAndroidPusher) 

  使用 Mediacodec 硬编码 H.264以及AAC 音视频，通过simplertmp推到 rtmp server
  采用opengl 硬编码码流获取方式
  将yasea项目中有关rtmp的部分单独玻璃封装，方便扩展
  platform: API 15及以上	
  硬编码
  
* [rtmp-rtsp-stream-client-java](https://github.com/pedroSG94/rtmp-rtsp-stream-client-java)

#### 美颜

* [android-gpuimage](https://github.com/CyberAgent/android-gpuimage)
  
  Android 2.2 or higher (OpenGL ES 2.0)
  
* [Grafika](https://github.com/google/grafika)
  
  An SDK app, developed for API 18 (Android 4.3)
  
#### 开源SDK

* [FREYA-STREAM-CASTER-SDK-FOR-ANDROID](https://github.com/jkkj93/FREYA-STREAM-CASTER-SDK-FOR-ANDROID) 推荐
  ANDROID 4.0.3及以上
  JNI层开源

* [Kickflip](https://github.com/Kickflip/kickflip-android-sdk) 坑多慎入
  
  Android 4.3+ (API 18+) 
  

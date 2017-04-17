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
  
  同时定义一个单线程线程池，来达到队列执行线程任务目的。
  
  ```java
  
  ExecutorService gifNotifyQueueService = Executors.newSingleThreadExecutor
  
  ```
  
  在RevGifQueue的构造函数中，开启单线程任务队列的执行。
  
  ```java
  
  private RevGifQueue() {
     Log.i(TAG, "RevGifQueue");
     giftNotifyQueue = new LinkedBlockingQueue<UserGiveGiftNotify>();
     gifNotifyQueueService.execute(new GetGifDataRunnable());
  }
  
  
  class GetGifDataRunnable implements Runnable {
        @Override
        public void run() {

            while (isStart) {
                try {
                    if (giftNotifyQueue != null && giftNotifyQueue.size() == 0) {
                        SystemClock.sleep(5);
                        continue;
                    } else {
                        if (giftNotifyQueue != null) {
                            // 每隔50毫秒在队列取一个礼物
                            SystemClock.sleep(50);
                            UserGiveGiftNotify gifNotify = giftNotifyQueue.poll(1,TimeUnit.MILLISECONDS);
                            if (null != gifNotify) {
                                UserGiveGiftNotifyEvent notify = new UserGiveGiftNotifyEvent();
                                notify.setNotify(gifNotify);
                                EventBus.getDefault().post(notify);
                            }
                            continue;
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
  ```
  
  LinkedBlockingQueue可以提供无界阻塞队列的功能，其offer/poll方法分别向队列写入/读取数据，如果队列为空，那么poll执行时将进入阻塞状态，直到LinkedBlockingQueue的数据不为空位置。
  
  客户端同前后端通过基于MINA2的网络框架进行数据通信，在接收到服务器推送的UserGiveGiftNotify后，会将该nofity写入LinkedBlockingQueue。
  
  ```java
  
  //将LinkedBlockingQueue设置为有界
  if (giftNotifyQueue.size() <= QUEUE_MAX_SIZE) {
      giftNotifyQueue.offer(giftNotify);
  }
  
  ```
  
  GetGifDataRunnable线程每隔50毫秒poll一次giftNotifyQueue，当有数据可取时，通过EventBus全局post一个UserGiveGiftNotifyEvent，在其他UI线程可以接收
UserGiveGiftNotifyEvent事件的地方即可执动画播放任务。

最后，在进入直播间时，初始化RevGifQueue，在推出直播间时，release掉RevGifQueue，整个流程就基本搞定了。

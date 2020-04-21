---
layout: post
title: '直播架构、编码'
subtitle: '编码、H264、直播'
date: 2018-12-03
categories: 技术
cover: 
tags: AVFoundation
---



### App直播流程

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-01.jpg)

* 采集视频、音频
    * 摄像头、麦克风
        * 摄像头 CDD(图像传感器)
        * 麦克风、拾音器(声音传感器)
    * iOS采集音视频数据
        * 导入AVFoundation.Framework框架
        * 从CaptureSession会话的回调中获取音视频数据
* 视频处理、滤镜
    * 美颜
    * 水印
    * 使用CPUImage(已包含采集操作)
* 视频、音频编码压缩
    * 硬编码
        * 视频：VideoToolBox框架(iOS8.0)
        * 音频：AudioToolBox框架(iOS8.0)
    * 软编码
        * 视频压缩
            * 视频编码MPEG、H264
            * X264把视频原数据YUV/RGB编码H264
        * 音频压缩
            * 音频编码：mp3、AAC 
            * fdk_aac将音频数据PCM转AAC
* 推流
    * 什么是推流？将采集的音频、视频数据通过流媒体协议发送到流媒体服务器
    * muxing(封装)音视频封包成FLV或者TS
    * 推流技术
        * 流媒体协议：RTMP、RTSP、HLS、FLV
        * 视频封装格式：TS、FLV
        * 音频封装格式：mp3、AAC
* 流媒体服务器
    * 数据分发(CDN)
    * 截屏
    * 录制
    * 实时转码
* 拉流
    * 什么是拉流？从流媒体服务器获取音频、视频数据
    * 流媒体协议：RTMP、RTSP、HLS、FLV
* 音视频解码
    * demuxing(解封装)
        * 将FLV、TS文件分离出音视频
    * 视频解码
        * 硬编码VideoToolBox
        * 软解码X264
    * 音频解码
        * 硬解码AudioToolBox
        * 软解码fdk_aac
* 音视频播放
    * ijkplayer:一款开源强大的播放器，基于FFmpeg封装
* 聊天互动
    * IM即时通讯
    * 聊天室功能
    * 第三方即时通讯：融云、腾讯云

##### 直播架构

![直播架构](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-02.jpg)

##### 小视频架构

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-06.jpg)

##### 封⾯处理!
 * 美拍/秒拍/抖⾳音/直播 
 * 录制选择一个封面!
    * 服务器分发给你—直播(支持回放!)
    * 客户端:默认第一帧视频就是封⾯—⼩视频 
    * 客户端:从视频中选择合适的一帧作为封面—小视频

### 直播产品分类
* 泛娱乐直播(斗鱼，游戏)
* 时时互动直播(教育类、视频会议，延时不能过长)

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-03.jpg)

主播端向信令服务器发请求，服务器接受到后开设直播间，把CDN流媒体的地址返回主播端，主播端拿到地址后不断的向CDN发送直播数据。观众向信令服务器发请求，从信令服务器获取房间地址，就是CDN的地址，不断从CDN获取数据

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-04.jpg)

受传输协议影响。保证稳定健壮，使用多个信令服务器。如何保证负载均衡？每个节点(信令服务器)定期会向控制中心汇报自己的健康指数(CPU占用、内存占用、IO占用、网络占用)，如果出现某一个数据不达标时，就会把服务器上面的任务切到其他节点上，当检测到占用过高，分配给一个比较闲的节点，中间层也叫心跳线。媒体服务器将实时互动网络和泛娱乐网络做转换

#### CDN网络解析

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-05.jpg)

CDN是为了解决用户在网络资源慢的时候一种技术。为什么会慢？1.链路过长(北京--长沙)；2.行业竞争人为因素(南电信北联通)
边缘节点：在附近找的了资源。比如北京烤鸭，长沙有分店，所以不需要去北京了
二级节点：在张家界附近没有找到边缘节点，在长沙找到了。会把数据传给边缘节点，边缘节点再给用户
源站节点：内容提供方

### IBP帧以及视频花屏卡顿原因分析

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-07.jpg)

* 内容元素
    * 图像
    * 音频
    * 元信息
* 编码格式
    * Video：H264
    * Audio：AAC
* 容器封装
    * MP4、MOV、FLV、RM、RMVB、AVI


![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-09.jpg)

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20181203-10.jpg)


* I帧：关键帧，一组图片中为了减少存储会保存一张完整的图片，第一张
* P帧：向前参考帧，只会保存跟前一张不一样的数据，没有就不保存
* B帧：双向参考帧，参考前一帧和后一帧然后保存不一样的数据

>如果在解码时丢失了I帧没有办法正确解码....
>没有B帧，只有I帧P帧也可以做压缩。丢弃B帧，并不会降低存储量，因为在使用B帧的时候，是向前和向后参考，网络传输的时候解码B帧，前面一帧出来了向前参考，后面一帧出来了才可以解码B帧，等待前后两帧传输完毕才可以解码B帧，此时解码B帧需要等待时间，没有办法断定网络传输就一定很快，I帧B帧P帧不是一次性传过来的，需要等待！如果是实时网络互动，因为有这样的等待，会造成直播的延时性，所以说在实时互动要求延时性非常低，时效性的时候会丢弃B帧
>在泛娱乐的时候需要B帧，因为B帧压缩性更高，对于存储和传输有优势

### 帧内预测压缩和帧间预测压缩原理分析

* 帧内预测压缩，解决的是空域数据冗余问题，I帧
    * 什么是空域数据？就是这幅图里数据在宽高空间内包含了很多颜色、光亮，人的肉眼很难察觉的数据，对于这些数据，我们可以认作冗余，直接压缩掉的，有损压缩！
* 帧间预测压缩，解决的是时域数据冗余问题，P帧
    * 摄像头在一段时间内所捕捉的数据没有较大的变化，我们针对这一时间内的相同数据压缩掉，这叫时域数据压缩
* 整数离散余弦变换(DCT)，将空间上的相关性变为频域上无关的数据然后进行量化
    * 这个比较抽象，这个跟数学是紧密联系在一起的，如果对傅里叶变换理解的比较好的，对这个理解的会比较快。傅里叶变换可以把一个复杂波形图变换成许多对正弦波，只是他们之间的频率不一样，以及振幅不一样，如果他们在频率上没有一致性那么我们就可以对它进行压缩处理
* CABAC压缩：无损压缩


### 视频封装格式

* 为什么不用H265
    * iOS11后才支持H265 
    * CPU发热


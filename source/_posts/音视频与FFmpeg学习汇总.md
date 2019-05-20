---
title: 音视频中的基础概念
date: 2019-05-15 15:37:04
tags: [音视频]  
categories: Android
---

音视频学习笔记第一篇~

<!-- more -->  



# 格式

## 颜色（像素）格式

详见参考

### RGB

作为android开发最熟悉的就是rgb格式了，

在RGB颜色空间中，任意色光F都可以用R、G、B三色不同分量的相加混合而成：F=r[R]+r[G]+r[B]

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/rgb%E9%A2%9C%E8%89%B2%E6%A8%A1%E5%9E%8B.jpeg?raw=true)

### HSV

HSV是一种将RGB色彩空间中的点在倒圆锥体中的表示方法。HSV即色调(Hue)、饱和度(Saturation)、明度(Value)，又称HSB(B即Brightness)。

色调H：一些基础颜色

饱和度S：用过ps就很好理解这个了，我理解的话是色彩的鲜艳程度，S=0就是灰度化

明度V：就是亮度了，看右图就很好理解了，V=0就是暗的一B，黑色，V=max，就是亮瞎狗眼，白色。

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/hsv颜色模型.jpeg?raw=true)

### YUV

YUV则是把颜色用一种更直接抽象的方式来表达，“Y”表示明亮度，“U”和“V” 表示的则是色度。Y是通过RGB信号特定部分**叠加**得到的，UV则用来反应颜色的**色调**和**饱和度**。分别用Cr和Cb来表示。其中，Cr反映了RGB输入信号红色部分与RGB信号亮度值之间的差异。而Cb反映的是RGB输入信号蓝色部分与RGB信号亮度值之间的差异。

采用YUV色彩空间的重要性是它的亮度信号Y和色度信号U、V是分离的。如果只有Y信号分量而没有U、V分量，那么这样表示的图像就是黑白。黑白电视只利用Y分量，也解决了黑白电视和彩色电视的兼容问题。

有公式可以转换为RGB，上述的`NV21`算是它的子集了。

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/yuv%E9%A2%9C%E8%89%B2%E6%A0%BC%E5%BC%8F1.png?raw=true)

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/yuv%E9%A2%9C%E8%89%B2%E6%A0%BC%E5%BC%8F2.png?raw=true)



## 视频的编码格式

视频我们都知道，是由一帧又一帧的图片连续显示组成的。

假设有个帧率为20的1080*720分辨率的视频，1秒播放20张图片，每一帧图片用的是RGB565格式，即一个像素点占16bit。那么根据上面已知条件，可以计算出想要在线播放这段视频时需要的网速应该要 `20x1080x720x16/8 =31104000 `，即29M/s，显然不现实，更别说服务器要花多少空间来存这个视频了。

所以就出现了一系列的算法标准和规范，来压缩视频大小。

![偷图小冬瓜又来了](https://github.com/MelonWXD/BlogPicStorage/blob/master/%E8%A7%86%E9%A2%91%E7%BC%96%E7%A0%81%E6%A0%BC%E5%BC%8F.png?raw=true)

### H.264

> 注意区别 
>
> MPEG-4是一整套的音频，视频，编解码标准的集合
>
> H.264是一个国际标准的视频编码压缩格式 它跟MPEG-4的视频压缩编码是一致的 是MPEG-4的第10部分

具体的Data Struct详见参考。这里只是简单说一下基础的概念。

视频帧经过算法压缩后的帧分为：I帧，P帧和B帧:

I帧：关键帧，采用帧内压缩技术。

P帧：向前参考帧，在压缩时，只参考前面已经处理的帧。采用帧音压缩技术。 

B帧：双向参考帧，在压缩时，它即参考前而的帧，又参考它后面的帧。采用帧间压缩技术。

GOP:两个I帧之间是一个图像序列，在一个图像序列中只有一个I帧。

我的理解是他存储的是每一帧指间的差异，因为视频嘛，图片都是具有连贯性的。而关键帧I帧就是定下基调的一帧，对应的GOP中都是基于I帧不断修修补补的，编码时发现无法恢复信息时，就又会生成一个I帧，结束当前GOP。

### 其他的压缩格式 ：VP9 ...

了解的不多 待补充

## 视频的封装格式

我们常见的**音视频**文件格式，诸如avi、mp4和rmvb都是封装格式。

跟上面编码格式的区别是什么？我觉得最大的就是封装格式里面含有音频流信息呗。

### MP4

详见参考

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/MP4%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F.jpg?raw=true)



这种文件格式的解析，基本套路都是一样的。



## 音频的编码格式

音频常见的压缩格式为aac mp3



# Android中的音视频操作

了解完格式，让我们描述一个完整的android视频操作：相机录像。

## 相机录像

根据 `android/hardware/Camera.java` 内部类Parameters的setPreviewFormat方法注释可以知道

```java
/**
 * Sets the image format for preview pictures.
 * <p>If this is never called, the default format will be
 * {@link android.graphics.ImageFormat#NV21}, which
 * uses the NV21 encoding format.</p>
 ...
 */
public void setPreviewFormat(int pixel_format) {...}

```

默认情况下 相机预览得到的照片的**颜色格式**是`NV21`，[这里](<https://developer.android.com/reference/android/graphics/ImageFormat.html>)可以看到android中支持的所有ImageFormat。

系统提供一个`YuvImage`的类，支持把`NV21`压缩成`JPEG`格式的图片，这个可能是我们比较熟悉的图片格式。

> 注意
>
> 这里`NV21`是**颜色格式**，属于`YUV420`类别的，同级别的比较应该是`HSV`与`RGB`。而`JPEG`是**图片格式**，同级别比较的是`PNG`、`BMP`这种的。

 我们是可以在`Camera.onPreviewFrame(byte[] bytes, Camera camera)`回调方法中拿到这些数据

下面就要用到Android的编码类`MediaCodec` 对数据帧进行压缩编码

### MediaCodec 

典型的生产消费模型。

编解码器可以处理三种类型的数据：

1. 压缩数据（即为经过H254. H265. 等编码的视频数据或AAC等编码的音频数据）
2. 原始音频数据
3. 原始视频数据

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/mediacodec.png?raw=true)

### MediaMuxer

将音频编码和视频编码合成为封装格式(.mp4 .avi)的Android api类

相应的还有**MediaExtractor**来负责解封装  从封装格式中提取音频流和视频流

音频的获取主要依赖**AudioRecord**、**AudioTrack**  就不展开了细说了 跟视频一个意思



## 相机录像流程图

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%AD%A6%E4%B9%A0-%E5%BD%95%E5%88%B6-%E6%92%AD%E6%94%BE.jpg?raw=true)

注意，这个过程是可逆的，反过来就是安卓如何播放一个mp4了，合成器变分离器，编码器变解码器，输出到Surface上显示和AudioManager里播放就是了。至于音频的采集和编码就不赘述了，一个道理。

## 其他

### FFmpeg

Android在MediaCodec出现之前使用的编解码工具，用c写的，通过so库调用，软编码，兼容性好，性能略低。

### openGL ES

这里水深，后续有机会再扩展。

据我了解的在安卓中的应用应该是滤镜。

应用时机应该是在YUV格式阶段。





# 参考

[常见视频编码格式解析](https://blog.csdn.net/houxiaoni01/article/details/78812485/)

[颜色空间模型](https://www.cnblogs.com/yooyoo/p/4717746.html)

[H264 基本原理](https://blog.csdn.net/garrylea/article/details/78536775)

[H264 结构](https://juejin.im/post/5a8fe66b6fb9a0633e51eadc)

[mp4结构](<https://www.cnblogs.com/ranson7zop/p/7889272.html>)
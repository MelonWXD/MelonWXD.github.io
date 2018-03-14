---
title: Android性能优化 
date: 2018-03-14 11:24:55
tags: [性能优化]  
categories: Android
---
Android性能优化简洁
<!-- more -->  

# 布局优化
## 布局标签的使用
### include

通过include将共同的布局模块化，实现布局复用

### merge

merge会自动优化多余的视图，多用于替换frameLayout或者当一个布局包含另一个布局的时候。

**当把有`<merge>`标签的布局放在`<include>`中的时候，就会忽视`<merge>`**

### viewstub

viewstub常用来引入那些默认不会显示，只在特殊情况下显示的布局，如进度布局、网络失败显示的刷新布局、信息出错出现的提示布局等。

##　其他

- **用SurfaceView或TextureView代替普通View**

-　**使用OpenGL绘图**

-　**尽量为所有分辨率创建资源**

-　**hierarchy viewer布局调优**

  ​

# 代码优化

## 降低执行时间

### 缓存

- 线程池
- 图片缓存：内存、磁盘
- handler消息复用
- Http的Cache-Control
- IO缓存：BufferedInputStream替代InputStream，BufferedReader替代Reader，BufferedReader替代BufferedInputStream.对文件、网络IO皆适用。

### 数据选择

- String、StringBuilder、StringBuffer
- SoftReference、WeakReference
- final类型存储在常量区中读取效率更高
- 应用内广播LocalBroadcastManager 高效、安全
- ArrayList和LinkedList，HashMap和LinkedHashMap、Set
- 使用自带的SpareArray和ArrayMap 替换HashMap

## 提高执行效率

- 多线程
- 延迟操作 postDelay

## 网络优化

- 所有http请求必须添加httptimeout
- gzip压缩  请求合并



# 参考

[性能优化系列总篇](http://www.trinea.cn/android/performance/)

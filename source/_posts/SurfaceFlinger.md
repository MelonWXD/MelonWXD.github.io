![](https://source.android.google.cn/devices/graphics/images/bufferqueue.png?hl=zh-cn)


# Gralloc的加载
## BufferNativeWindow
fb打开的驱动信息存储在fbDev
` err = framebuffer_open(module, &fbDev);`
gralloc打开的信息在grDev
` err = gralloc_open(module, &grDev);`
fbDev负责的是主屏幕，grDev负责图形缓冲去的分配和释放。

>每一个bufferqueue对应的都是一个layer

## 双重缓存 三重缓存
双重缓存如果在VSync到来之前GPU没有绘制完毕 显示器就继续保持上一帧 就不会不释放上一个buffer CPU和GPU在超时之后绘制完毕 就没事干了 
如果三重缓存的话 就会多余一个buffer来让CPU和GPU提前绘制 相当于说要最多可以等2个VSync才会轮到这个buffer的显示



## BufferQueue的生产者/消费者模型

在进入讨论这些扩展之前，先简单回顾下Andriod BufferQueue的运行机制。

在Android (3.0之后)，上到application,下到surfaceflinger, 所有的绘制过程都采用OpenGL ES来完成。对于每个绘制者（生产者，内容产生者）来说，步骤大致都是一样的。

(1)获得一个Surface（一般是通过SurfaceControl）  
(2)以这个Surface为参数，得到egl draw surface 和 read surface. 通常这俩是同一个对象  
(3)配置egl config,然后调用eglMakeCurrent()配置当前的绘图上下文。  
(4)开始画内容，一般通过glDrawArray()或者glDrawElemens()进行绘制  
(5)调用eglSwapBuffers() swap back buffer和front buffer。  
(6)回到第4步，画下一帧。  


### Producer  可以认为是App
### Consumer 可以认为是SurfaceFlinger
每当用户程序刷新UI的时候，会向BufferQueue申请一个buffer（dequeueBuffer），然后把UI的信息填入，丢给SurfaceFlinger，SurfaceFlinger通过计算多重计算合成visibleRegion之后，丢给openGL层处理，处理之后送到显示器display上显示。
### 图形显示过程
1. startActivity启动Activity；
2. 为Activity创建一个window(PhoneWindow)，并在WindowManagerService中注册这个window；
3. 切换到前台显示时，WindowManagerService会要求SurfaceFlinger为这个window创建一个surface用来绘图。SurfaceFlinger创建一个”layer”（surface）。（以想象一下C/S架构，SF对应Server，对应Layer；App对应Client，对应Surface）,这个layer的核心即是一个BufferQueue，这时候app就可以在这个layer上render了；
4. 将所有的layer进行合成，显示到屏幕上。

![](http://o7xxrho8u.bkt.clouddn.com/file/windrunnerlihuan%E5%8D%9A%E5%AE%A2/Android%20SurfaceFlinger%20%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF%28%E4%BA%8C%29----SurfaceFlinger%E6%A6%82%E8%BF%B0/gui.jpg)

### 窗口的持有
根据整个Android系统的GUI设计理念，我们不难猜想到至少需要两种本地窗口：

- 面向管理者(SurfaceFlinger)：既然SurfaceFlinger扮演了系统中所有UI界面的管理者，那么它无可厚非地需要直接或间接地持有“本地窗口”，这个窗口就是FramebufferNativeWindow
- 面向应用程序：这类窗口是Surface（这里和以前版本出入比较大，之前的版本本地窗口是SurfaceTextureClient）

![](http://img.blog.csdn.net/20160117163815330)
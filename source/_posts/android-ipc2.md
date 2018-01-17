---
title: 浅析Android进程间通信（二）
date: 2017-10-28 15:10:03
tags: [IPC]
categories: Android
---
结合Linux下IPC机制来简单分析Android底层的Binder机制
<!-- more -->
# Linux系统的进程间通信
## 信号
信号是Unix/Linux系统在一定条件下生成的事件。信号是一种异步通信机制，信号会通知 接收信号的进程 发生了某个事件，然后操作系统将会中断该进程的执行，跳转去执行相应的信号处理程序。  
听起来像什么？想不想硬件上面电信号对CPU的中断，一个意思的，就是信号这个是软件层模拟的对进程的一种中断形式，深入我就没了解了，有需要自己去百度。
## 管道
通过管道来实现IPC主要有3种
- popen()和pclose()函数
- pipe()函数
- 命名管道FIFO

### popen() pclose()
```c
#include <stdio.h>
FILE *popen(const char *cmdstring, const char *type);
返回值：若成功则返回文件指针，若出错则返回NULL

int pclose(FILE *fp);
返回值：cmdstring的终止状态，若出错则返回-1
```
popen()函数的返回值是一个普通的标准I/O流, 它只能用 pclose() 函数来关闭/如果type是'w'，则返回的文件指针是可写的，通过这个流的写入的内容会被转化为shell命令的标准输入（就是会让读这个管道的进程执行相应的shell命令？）  
fp = popen（cmdstring， "w"）：  
![](http://images.cnitblog.com/blog/468825/201402/221953109902246.jpg)  
fp = popen（cmdstring， "r"）   
![](http://images.cnitblog.com/blog/468825/201402/221953097004287.jpg)
### pipe()
pipe()函数可以返回两个管道描述符:pipefd[0]和pipefd[1]，任何写入pipefd[1]的数据都可以从pipefd[0]读回；pipe()函数返回的是文件描述符(file descriptor)，因此只能使用底层的read()和write()系统调用来访问。pipe()函数通常用来实现父子进程之间的通信：
 ![](http://img.blog.csdn.net/20161223173958916)

### FIFO
终于到我大FIFO了，有没有发现前面的2种方法只能用在相关的进程中，如果2个啥关系也没有的进程怎么通过一样的原理（一个写，一个读）来实现IPC呢。  
这就要用到命名管道FIFO了，让2个进程都打开同一个文件（特殊文件），来实现数据交换  
 写：
 ```
   /* open函数以O_WRONLY方式打开FIFO文件,如果成功pipe_fd指向打开的文件 */  
    pipe_fd = open(FIFO_NAME, O_WRONLY);  
    /* 再使用write函数来把buffer中的内容写到fifo文件中，成功则返回实际写入的字节数 */  
    res = write(pipe_fd, buffer, BUFFER_SIZE);  
 ```
读：
```
    /* open函数打开FIFO_NAME文件,以O_RDONLY的方式(即只读) 
     * 如果成功,则返回文件描述符 */  
    pipe_fd = open(FIFO_NAME, O_RDONLY);  
    /* 再使用read函数来读取fifo文件中的内容到buffer中 */  
    res = read(pipe_fd, buffer, BUFFER_SIZE);  
```
读写之间的阻塞操作是由系统的来调度的，具体参考参考中的FIFO读写规则。

# Android系统的进程间通信
ok，终于把linux的ipc大致理了一下，为什么在说Android的进程间通信之前，要先把linux的下面的说一下？因为Android是基于Linux的，实现的思路肯定也会有参考原有的思想，还有就是我也是刚学，多写一写记得牢一些。。  
关于Android系统下的IPC我不打算像网上那样贴一大堆代码，只见树木不见森林，我就说几句话，毕竟是做笔记成分多一点，能不能懂就看个人了  
Android系统的IPC包括以下4个主要项：
- Binder驱动
- ServiceManager
- Server(Remote Service)
- Client

套用一句别人的话：
> Binder框架定义了四个角色：Server，Client，ServiceManager以及Binder驱动。其中Server，Client，ServiceManager运行于用户空间，Binder驱动运行于内核空间。这四个角色的关系和互联网类似：Server是服务器，Client是客户终端，ServiceManager是域名服务器（DNS），Binder驱动是路由器。  


## Binder驱动
工作于内核态，提供open()，mmap()，poll()，ioctl()等标准文件操作。  
一句话，他就相当于管道，ServiceManager、Server和Client实际上都是通过Binder驱动和Binder来打交道，是Android进程间通信的基础桥梁，封装了具体的调用细节。  
注意区分：Binder驱动和Binder，Binder驱动是Android IPC机制的底层实现，Binder是一个封装了读写细节的类，所有远程服务都要继承Binder。可以理解为1个Binder就是1个服务，他们都是由Binder驱动来调用操作的。
## ServiceManager
Service会创建了多个Binder的实体，对应着不同的Remote Service（PowerManagerService、ActivityManagerService等，全都继承于Binder，否则怎么能够在Binder驱动中实现通信呢），会向ServiceManager注册，传给ServiceManager需要注册的Binder的名字和引用。作用确实类似于DNS，存放着 名字(域名) 和 引用（ip地址）。  
但是有没有想过，ServiceManager和Service在不同的进程，怎么实现这两者之间的进程通信呢？  
所以就有了一个叫做0号Binder的东西，就好比你知道114.114.114.114是国内的通用DNS一样，他们定死了一个Binder，来让其他进程直接通过Binder驱动来默认使用这个0号Binder实现与ServiceManager的进程间通信。  
## Server
系统服务，通过Binder驱动调用0号Binder向ServiceManager注册多个系统服务，比如底层的系统电量管理服务，内容（假设）为："name = PowerManagerService，addr = 0x001"。那么在ServiceManager中就会记录下来，远程服务PowerManagerService的Binder引用为0x001。
## Client
客户端想要调用某个服务时，也要先通过Binder驱动调用0号Binder来向ServiceManager请求，如果请求的PowerManagerService，那么ServiceManager就会返回PowerManagerService的引用给他，Client就拿到PowerManagerService的Binder引用地址为0x001。然后再通过Binder驱动，与PowerManagerService进行数据交换。

## 总结
每个远程服务，都是Binder的子类，继承于Binder，使得他们可以基于Binder驱动进行数据读写。可以把Binder认为是每个远程服务派到当前调用服务的用户进程的代表（代理），有什么事，跟我的代表说就行了。通过我的代表来调用我的方法。  
一次调用系统电量服务，查询电量值的流程如下：  
系统启动，Server向Binder驱动写入信息流：`使用0号Binder 注册name=PowerManagerService addr=0x001`，那么Binder就会解析，然后找到并调用0号Binder，对0号Binder写入`注册name=PowerManagerService addr=0x001`，然后0号Binder就会调用ServiceManager的方法，注册PowerManagerService。  
Client向Binder驱动写入信息流`使用0号Binder 查找name=PowerManagerService`，那么Binder就会解析，然后找到并调用0号Binder，向ServiceManager请求得到PowerManagerService的引用地址。然后Binder驱动就把得到的`引用地址=0x001`返回给Client。
Client再通过向Binder驱动写入请求信息流`使用地址0x001的Binder 执行命令 getPowerValue`，Binder驱动就会解析，然后找到并调用地址为0x001号Binder，写入命令`执行命令 getPowerValue`，那么 `0x001`所对应的Binder就会对他对应的PowerManagerService调用getPowerValue，再把得到的值返回给Binder驱动，Binder驱动再返回给Client，结束调用。

**那么问题来了，Server和Client是怎么和Binder驱动进行通信的呢？**

是通过内存映射，让驱动设备文件映射到用户进程和内核进程，即用户进程地址V1和内核进程地址V2都映射着同一块物理内存，那么用户把进程把数据写到文件上，内核进程自然就能读取到内容，而不需要在copy到内核去。具体查看[binder驱动-------之内存映射篇](http://blog.csdn.net/xiaojsj111/article/details/31422175)

![](http://hi.csdn.net/attachment/201102/27/0_1298798582y7c5.gif)

# 参考
[popen()和pclose()](http://www.cnblogs.com/nufangrensheng/p/3561190.html)  
[Linux命名管道FIFO的读写规则](http://blog.csdn.net/MONKEY_D_MENG/article/details/5570468)  
[Android Binder设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)
---
title: Unix I/O多路复用
date: 2017-12-30 15:46:49
tags: [并发]
categories: CSAPP
---
分析select函数来学习 I/O多路复用
<!-- more -->



# 什么是 I/O多路复用

IO 多路复用是5种I/O模型之一（阻塞IO 非阻塞IO IO复用 信号驱动IO 异步IO）

## 阻塞IO

当服务器要read一个socket链接的时候，内核挂起该线程，直到改socket有数据写入

当服务器要处理1000个并发的时候，需要1000个线程，内存开销大

## 非阻塞IO

当服务器要read一个socket链接的时候，没有数据立即返回-1或返回一个错误

服务器需要不断轮询

## IO多路复用

用一个线程，来管理一堆文件描述符fd，不断的轮询检查是否有fd可读（或者可写），如果有，就返回，否则一直阻塞到超时。

得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行（比如使用线程池）。

这样在处理1000个连接时，只需要1个线程监控就绪状态，对就绪的每个连接开一个线程处理就可以了，这样需要的线程数大大减少，减少了内存开销和上下文切换的CPU开销

# Select()
```c
	int select(int maxfdp,fd_set *readfds,fd_set *writefds,fd_set *errorfds,struct timeval *timeout);
```




## fd_set

结构体如下

```c
#define __NFDBITS (8 * sizeof(unsigned long))                //每个ulong型可以表示多少个bit,
#define __FD_SETSIZE 1024                                          //socket最大取值为1024
#define __FDSET_LONGS (__FD_SETSIZE/__NFDBITS)     //bitmap一共有1024个bit，共需要多少个ulong
 
typedef struct {
    unsigned long fds_bits [__FDSET_LONGS];                 //用ulong数组来表示bitmap
} __kernel_fd_set;
```

fd_set这个结构实际上就是一个数组，每个数组按位存放fd，即文件描述符，提供了4个宏定义来让我们操作fd_set

```
(1)、FD_ZERO(fd_set *)  清空一个文件描述符集合；
(2)、FD_SET(int ,fd_set *) 将一个文件描述符添加到一个指定的文件描述符集合中；
(3)、FD_CLR(int ,fd_set*)  将一个给定的文件描述符从集合中删除；
(4)、FD_ISSET(int ,fd_set* ) 检查集合中指定的文件描述符是否可以读写。
```

那么参数中的`fd_set *readfds,fd_set *writefds,fd_set *errorfds`也非常好理解了：

- readfds：这个数组中包含了所有我们要关注的fd的可读状态，如果这个数组中的某个fd状态变为可读，那么select就返回一个正整数，表示某个或多个文件目前可读
- writefds：同上，可写
- errorfds：同上，异常

附图一张加深理解：

![](http://img.blog.csdn.net/20131007164314234?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGluZ2Zlbmd0ZW5nZmVp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



可以看到当程序开始监听的时候，通过宏定义来清空fd_set后再设置我们所关心的fd进去，在没有返回之后，通过`FD_ISSET`来找到变化的fd，判断是accept还是recv，然后再根据不同的操作还会在修改fd_set的某些位。



## timeout

结构体如下

```c
struct timeval{      
        long tv_sec;   /*秒 */
        long tv_usec;  /*微秒 */   
    }
```

有三种情况：

- timeout == NULL  等待无限长的时间。等待可以被一个信号中断。当有一个描述符做好准备或者是捕获到一个信号时函数会返回。如果捕获到一个信号， select函数将返回-1,并将变量 erro设为 EINTR。
- timeout->tv_sec == 0 && timeout->tv_usec == 0 不等待，直接返回。加入描述符集的描述符都会被测试，并且返回满足要求的描述符的个数。这种方法通过轮询，无阻塞地获得了多个文件描述符状态。
- timeout->tv_sec !=0 ||timeout->tv_usec!= 0 等待指定的时间。当有描述符符合条件或者超过超时时间的话，函数返回。在超时时间即将用完但又没有描述符合条件的话，返回 0。等待也会被信号所中断。






## 缺点

- select 会修改传入的参数数组，这个对于一个需要调用很多次的函数，是非常不友好的。
- select 当fd变化，select只是返回一个值，具体是哪个值还要我们去找，如果fd_set很大的话每次开销也很大
- select 只能监视1024个链接 
- select 不是线程安全的

基于以上考量，就有了poll这方法，但是poll也不是线程安全的，之后又有了epoll～



# 参考

[select详解](https://www.cnblogs.com/ccsccs/articles/4224253.html)

[知乎](https://www.zhihu.com/question/28594409)
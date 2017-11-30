---
title: Android 系统启动流程
date: 2017-11-11 11:11:11
tags: [源码分析]
categories: Android
---
Android系统启动流程分析
<!-- more -->
# Android系统启动过程分析
整个Android系统的启动分为Linux Kernel的启动和Android系统的启动。Linux Kernel启动起来后，然后运行第一个用户程序，在Android中就是init程序。Linux的启动也从其他文章copy到了本文后半段。
## init.cpp
源码路径:[/system/core/init/init.cpp](http://androidxref.com/6.0.1_r10/xref/system/core/init/init.cpp)
```
int main(int argc, char** argv) {
	...
       if (is_first_stage) {
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        mount("proc", "/proc", "proc", 0, NULL);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }
    ...
      if (!is_first_stage) {
         // Indicate that booting is in progress to background fw loaders, etc.
         close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
         property_init();
         ...
     } 
 
     start_property_service();
     init_parse_config_file("/init.rc");
     ...
}
```
主要做了以下工作：
- 在系统第一阶段(stage1)，挂载系统目录
- 在第二阶段(stage2)，初始化属性服务（[property_service](http://androidxref.com/6.0.1_r10/xref/system/core/init/property_service.cpp)）
- 分析并执行init.rc脚本

## init.rc
位置：`/system/core/rootdir/init.rc`
先来分析一下init.rc，这是android系统的启动配置文件，包括Android的启动第一个应用进程zygote的执行命令也是这个文件中。  
init.rc文件包含五个类型的声明：
- Action
- Commands
- Service
- Options
- Import
- 

# Linux系统启动过程分析
Linux系统启动流程图如下:  

![](http://images2015.cnblogs.com/blog/607348/201512/607348-20151229231206354-919070678.png)

## BIOS
BIOS(Basic Input/Output System)，基本输入输出系统，该系统存储于主板的ROM芯片上，计算机在开机时，会最先读取该系统。BIOS会按照启动顺序去查找第一个磁盘头的MBR信息，并加载和执行MBR中的Bootloader程序，若第一个磁盘不存在MBR，则会继续查找第二个磁盘，一旦BootLoader程序被检测并加载到内存中，BIOS就将控制权交接给了BootLoader程序。

## MBR
　　MBR(Master Boot Record)，主引导记录，MBR存储于磁盘的头部，大小为512bytes，其中，446bytes用于存储BootLoader程序，64bytes用于存储分区表信息，最后2bytes用于MBR的有效性检查。
### BootLoader
主要功能是初始化硬件设备、准备好软件环境，最后调用操作系统内核。  
![](https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=338703827,4227717080&fm=27&gp=0.jpg)
- Bootloader：上电后第一个程序。
- Boot parameters：分区中一般存放一些可设置的参数，比如IP地址、串口波特率，要传递给内核的命令行参数。
- kernel：嵌入式板定制的内核，包含内核启动参数
- Root filesystem文件系统：里面包含了linux能够运行的应用程序，和相关库等。


## GRUB
GRUB(Grand Unified Bootloader)，多系统启动程序，其执行过程可分为三个步骤：  
Stage1：这个其实就是MBR，它的主要工作就是查找并加载第二段Bootloader程序(stage2)，但系统在没启动时，MBR根本找不到文件系统，也就找不到stage2所存放的位置，因此，就有了stage1_5  
Stage1_5：该步骤就是为了识别文件系统  
Stage2：GRUB程序会根据/boot/grub/grub.conf文件查找Kernel的信息，然后开始加载Kernel程序，当Kernel程序被检测并在加载到内存中，GRUB就将控制权交接给了Kernel程序。
## Kernel
Kernel，内核，Kernel是Linux系统最主要的程序，实际上，Kernel的文件很小，只保留了最基本的模块，并以压缩的文件形式存储在硬盘中，当GRUB将Kernel读进内存，内存开始解压缩内核文件。讲内核启动，应该先讲下initrd这个文件，initrd(Initial RAM Disk)，它在stage2这个步骤就被拷贝到了内存中，这个文件是在安装系统时产生的，是一个临时的根文件系统(rootfs)。因为Kernel为了精简，只保留了最基本的模块，因此，Kernel上并没有各种硬件的驱动程序，也就无法识rootfs所在的设备，故产生了initrd这个文件，该文件装载了必要的驱动模块，当Kernel启动时，可以从initrd文件中装载驱动模块，直到挂载真正的rootfs，然后将initrd从内存中移除。

　　Kernel会以只读方式挂载根文件系统，当根文件系统被挂载后，开始装载第一个进程(用户空间的进程)，执行/sbin/init，之后就将控制权交接给了init程序。

## init
init，初始化，该程序就是进行OS初始化操作，实际上是根据/etc/inittab(定义了系统默认运行级别)设定的动作进行脚本的执行，第一个被执行的脚本为/etc/rc.d/rc.sysinit，这个是真正的OS初始化脚本

## Runlevel
runlevel，运行级别，不同的级别会启动的服务不一样，init会根据定义的级别去执行相应目录下的脚本，Linux的启动级别分为以下几种：  
　　0：关机模式  
　　1：单一用户模式(直接以管理员身份进入)  
　　2：多用户模式（无网络）  
　　3：多用户模式（命令行）  
　　4：保留  
　　5：多用户模式（图形界面）  
　　6：重启  












# 参考
[Linux 系统启动过程](https://www.cnblogs.com/codecc/p/boot.html)  
[SurfaceView入门学习](http://www.jianshu.com/p/15060fc9ef18)

---
title: Android 系统启动流程
date: 2017-12-08 15:48:11
tags: [源码分析]
categories: Android
---
Android系统启动流程分析，基于6.0源码
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

## 解析zygote

### init.rc
位置：[/system/core/rootdir/init.rc](http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.rc)
先来分析一下init.rc，这是android系统的启动配置文件，包括Android的启动第一个应用进程zygote的执行命令也是这个文件中。解析部分文件内容如下：

```
11import /init.${ro.zygote}.rc
12import /init.trace.rc
13
14 on early-init
15    # Set init and its forked children's oom_adj.
16    write /proc/1/oom_score_adj -1000
17
18    # Set the security context of /adb_keys if present.
19    restorecon /adb_keys
20
21    start ueventd
22
23 on init
24    sysclktz 0
```

值得注意的是跟之前的不同，`zygote`不再是直接写在文件中，而是通过`import`来导入，查看同级目录下有4个文件：

- [init.zygote32.rc](http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.zygote32.rc)   
- [init.zygote32_64.rc](http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.zygote32_64.rc) 
- [init.zygote64.rc](http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.zygote64.rc) 
- [init.zygote64_32.rc](http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.zygote64_32.rc) 

根据`ro.zygote`的值不同，来导入不同的文件，zygote64.rc内容：

```cpp
1 service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
2    class main    //声明service所属class的名字
3    socket zygote stream 660 root system
4    onrestart write /sys/android_power/request_state wake
5    onrestart write /sys/power/state on
6    onrestart restart media
7    onrestart restart netd
8    writepid /dev/cpuset/foreground/tasks
```



init.rc文件包含五个类型的声明：

- Action ：动作
- Commands ：命令
- Service ：服务
- Options ：服务的参数
- Import ：导入


结合[keywords.h](http://androidxref.com/6.0.1_r10/xref/system/core/init/keywords.h)可以查看文件中诸如`class_start`、`import`等关键字对应的命令类型。在头文件中`on`和`service`关键字被定义为了`SECTION`类型。



跟进上述init.cpp的`init_parse_config_file`方法，进入[init_parser.cpp](http://androidxref.com/6.0.1_r10/xref/system/core/init/init_parser.cpp#385)，在该方法内部实际调用了`parse_config`->` parse_new_section`，对zygote(service zygote)来说，主要是`parse_service`和`parse_line_service`两个方法。

```cpp
    case K_service:  
        state->context = parse_service(state, nargs, args);  
        if (state->context) {  
            state->parse_line = parse_line_service;  
            return;  
        }  
        break;  
```

### Service结构体

在具体查看解析代码之前，先来介绍一下[Service](http://androidxref.com/6.0.1_r10/xref/system/core/init/init.h#95)这个结构，因为最后文件的内容都要被解析成相关的信息，而Service就是存储这些信息的结构体。

```cpp
struct service {
	/*
		用来连接所有的services。Init中有一个全局的service_list变量，专门用来保存解析配置文件后得到的service。 
	*/
    struct listnode slist;

    const char *name;			// service的名字，在我们的例子中为zygote
    const char *classname;		// service所属的class名字，默认为default

    unsigned flags;			// service的属性
    pid_t pid;					// 进程号
    time_t time_started;    /* 上一次启动时间 */
    time_t time_crashed;    /* 上一次死亡时间*/
    int nr_crashed;         /* 死亡次数 */
    
    uid_t uid;
    gid_t gid;
    gid_t supp_gids[NR_SVC_SUPP_GIDS];
    size_t nr_supp_gids;

    char *seclabel;
	/*
		有些service需要使用socket来通信，下面这个socketinfo用来描述socket的相关信息。
		我们的zygote也使用了socket，配置文件中的内容是socket zygote stream 660 root system。
		它表示将创建一个AF_STREAM类型的socket（其实就是TCP socket），该socket的名为zygote，读写权限是660。
	*/
    struct socketinfo *sockets;
	// service一般运行在一个单独的进程中，envvars用来描述创建这个进程时所需的环境
	// 变量信息。
    struct svcenvinfo *envvars;
	/*
		虽然onrestart关键字是一个OPTION，可是这个OPTION后面一般都跟着一个COMMAND，action结构体就是用来存储command信息的。
	*/
    struct action onrestart;  /* Actions to execute on restart. */
    
    /* keycodes for triggering this service via /dev/keychord */
    int *keycodes;
    int nkeycodes;
    int keychord_id;

	// io优先级
    int ioprio_class;
    int ioprio_pri;

    int nargs; 	// 参数个数
    /* "MUST BE AT THE END OF THE STRUCT" */
    char *args[1];	// 用于存储参数内容
};
struct action {  
    /* 
        一个action结构可以被链入三个不同的双向链表中，其中alist用来存储所有的action， 
        qlist用来链接那些等待执行的action，tlist用来链接那些待某些条件满足后就需要执行的action。 
    */      
    /* node in list of all actions */  
        struct listnode alist;  
  
        /* node in the queue of pending actions */  
        struct listnode qlist;  
  
        /* node in list of actions for a trigger */  
        struct listnode tlist;  
  
        unsigned hash;  
        const char *name;   // 名字一般为"onrestart"  
      
    /* 
        一个command的结构列表，command结构如下： 
        struct command 
        { 
                /* list of commands in an action */  
                struct listnode clist;  
                int (*func)(int nargs, char **args);  
                int nargs;  
                char *args[1];  
        };  
    */  
        struct listnode commands;  
        struct command *current;  
};
```

![](http://img.blog.csdn.net/20140730144405770?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHUzMTY3MzQz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



## 启动zygote

init.rc中有一段是

```
403 on boot
...
495 on nonencrypted
496    class_start main
497    class_start late_start
```

`class_start`是一个`COMMAND`，`main`是类名，上述zygote文件中line2通过`class main`来把类名修改为`main`，所以这里就是启动zygote服务所属的类。

从[builtin.cpp](http://androidxref.com/6.0.1_r10/xref/system/core/init/builtins.cpp#112)的`do_class_start`一步步调用到[init.cpp](http://androidxref.com/6.0.1_r10/xref/system/core/init/init.cpp#service_start)的`service_start`方法。

```cpp
...
262    pid_t pid = fork();
354        if (!dynamic_args) {
355            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
...
357            }
358        } else {
...
374            execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
375        }
376        _exit(127);
...
```

通过`fork`以及调用`execve`来执行`/system/bin/app_process64`文件来创建zygote进程。



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
[Android启动分析](http://blog.csdn.net/hu3167343/article/details/38299969)

[Linux启动过程](https://www.cnblogs.com/codecc/p/boot.html)  
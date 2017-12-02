---
title: 注入之道(三) Inject和Hook
date: 2017-12-01 10:04:40   
tags: [注入] 
categories: Android逆向安全  
---

使用Inject来把so库注入进程，在目标进程加载So库来替换函数表
<!-- more -->

# Inject

## Ptrace
Ptrace是Unix/类Unix操作系统的一个系统调用，通过使用Ptrace可以让一个进程控制另一个进程，从而查看并修改的内部状态，通常是被用在debug或者代码分析等一系列辅助工具上。

> 下文中执行Ptrace的进程统称为控制进程，被控制进程、被注入进程统称目标进程

Ptrace也是注入程序Inject的核心功能支撑，通过Ptrace来读取、修改目标进程的寄存器，控制目标进程的挂起、执行，来达到我们想要的目的，即在目标进程中加载我们自己写的so库。

Ptrace的方法就一个，根据不同的request对目标进程pid做不同的操作。
```c

	// request: ptrace操作类型
	// pid_t  : 目标进程pid
	// addr   : 执行 peek 和 poke （内存读写）操作的目标地址
	// data   : 执行 poke 时作为数据写入到addr
	#include <sys/ptrace.h>
    long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```
下面来介绍一下基本的几个`__ptrace_request`：
- PTRACE_ATTACH ：attach到进程id为pid的进程
- PTRACE_DETACH ：detach
- PTRACE_GETREGS ：将目标进程寄存器值拷贝到 地址addr处
- PTRACE_SETREGS ：将控制进程 地址addr处的数据写入到 目标进程 寄存器中
- PTRACE_POKETEXT：将控制进程 地址data处数据 写入到 目标进程 地址addr处
- PTRACE_PEEKTEXT：读取目标进程 addr处数据 并返回
- PTRACE_CONT：让目标进程继续执行

## 整体流程
整个注入流程可以分为以下几点：
1.通过Ptrace Attach到目标进程
2.保存寄存器值（以便后续恢复现场）
3.调用mmap申请内存空间，来存放要加载的so库的路径
4.调用dlopen函数在目标进程中加载so库
5.[-]调用dlsym函数返回 so库中 某个方法名的地址 
6.[-]执行so库中的hook方法
7.[-]调用dlclose关闭动态链接库（只有动态链接库的使用计数为0时,才会真正被卸载）
8.恢复寄存器值
9.Ptrace Detach

其中567点不是必须的，如果so库的方法是so库的构造方法的话，形如：
```c
void __attribute__ ((constructor)) hooker_main() {
	...
}
```
在加载完so库之后链接器会自动调用so的构造方法，就不需要特地的再去找某个方法的地址，手动去调用了。

寄存器的保存和恢复比较简单，直接调用Ptrace的方法，下面主要讲一下方法地址的查找和执行。
## 确定方法地址
对于so文件来说，内部的函数一般分为导出函数和非导出函数，使用命令`nm -D xx.so`即可看到导出函数的偏移值，虽然每次加载so库的内存不同，但是如果知道so文件的内存基址base_addr，那么base_addr+偏移值就是某个导出函数的内存地址。
libc.so的导出函数mmap的偏移值如下：
```
duoyi@duoyi-OptiPlex-7010:~/Desktop$ nm -D libc.so | grep mmap
0000000000062a24 T mmap
0000000000062a24 T mmap64
```
既然偏移值是确定的，那么可以得出下面的式子：
`进程A中mmap函数地址 - 进程A的libc基址 = 进程B中mmap函数地址 - 进程B的libc基址`

基于这样一个式子，我们在控制进程中，主动加载libc库，取得mmap函数地址，记为`local_base_addr`和`local_method_addr`。

再通过遍历查询目标进程的内存映射文件，找到libc库在目标进程中的基址`remote_base_addr`，即可算出函数mmap地址`remote_base_addr`。目标进程的内存映射文件的位置为`/proc/pid/maps`，我这里用命令快速查看一下这样做的可行性，具体实现请看代码。

```
//权限
root@P635B32:/ # su
//查找目标进程surfaceflinger的pid=268 
root@P635B32:/ # ps | grep surf
system    268   1     307056 18256 ffffffff 829d8a4c S /system/bin/surfaceflinger
//查看目标进程的maps  过滤libc.so
root@P635B32:/ # cat /proc/268/maps | grep libc.so
7f82975000-7f82a25000 r-xp 00000000 b3:17 1959                           /system/lib64/libc.so
7f82a25000-7f82a2c000 r--p 000b0000 b3:17 1959                           /system/lib64/libc.so
7f82a2c000-7f82a2f000 rw-p 000b7000 b3:17 1959                           /system/lib64/libc.so
```
可以看到在进程`/system/bin/surfaceflinger`中libc.so的基址也是可以查到。

## 执行方法
先讲几个ARM中比较重要的几个寄存器：
- PC ：总是指向下一条指令
- SP ： 堆栈指针寄存器
- LR ：链接寄存器，用来保存子程序的返回值
- R0~R3 ：传参

结合上述的`__ptrace_request`，基本可以想到执行方案的思路：
使用`PTRACE_POKETEXT`修改寄存器PC值为要执行的方法地址`remote_method_add`。
但是这样一旦修改完，调用`PTRACE_CONT`，目标进程就一直执行下去了，也不知道方法是否执行完成，所以就用到了LR寄存器。
通过把LR寄存器手动置为0，当方法执行完成，跳转到LR寄存器时，目标进程尝试访问无效的内存引用，就会发生错误，进入暂停状态同时发送系统信号`SIGSEGV(#11)`。因此Ptrace就可以通过捕获这个错误，来达到既执行了方法，又没有丢失目标进程的控制的目的。
下面结合代码来说明：
```c
bool Tracer::traceCall(void *addr, t_long *params, uint8_t num_params, arch_regs *regs) {
    if (pid > 0) {
        //把参数存到寄存器中
        for (uint8_t i=0; i<num_params && i<MAX_PARAMETER_REGISTER; i++) 		{
            regs->uregs[i] = params[i];
        }
		//若参数个数超过MAX_PARAMETER_REGISTER 则把剩下的存到堆栈中
        if (num_params > MAX_PARAMETER_REGISTER) {
            size_t size = (num_params-MAX_PARAMETER_REGISTER)*sizeof(t_long);
            regs->sp -= size;
            traceWrite((uint8_t *) regs->sp, (uint8_t *) &params[MAX_PARAMETER_REGISTER], size);
        }
		//调整pc寄存器 地址
        regs->pc = (t_long)addr;
      
      	//确定指令集，调整cpsr状态寄存器
        if (regs->pc & 0x1) {
            // thumb
            regs->pc &= ~1u;
            regs->cpsr |= CPSR_T_MASK;
        } else {
            // arm
            regs->cpsr &= ~CPSR_T_MASK;
        }
      
        //设置lr寄存器地址为0
        regs->lr = 0;
		 
        if (traceSetRegs(regs) && traceContinue()) {
            int stat = 0;
            //WUNTRACED：当目标进程进入暂停状态时 waitpid返回
            //stat：低8位 表示目标进程是退出(0x0)还是暂停(0x7f)
          	//      高8位 表示导致退出或暂停的信号值
          	// 0b7f 就表示 因系统信号11 导致进程暂停
            waitpid(pid, &stat, WUNTRACED);
            while (stat != 0xb7f) {
                if (!traceContinue()) {
                    LOGE("failed to traceCall (%d)", pid);
                    return false;
                }
                waitpid(pid, &stat, WUNTRACED);
            }
            return true;
        }
    }
    LOGE("failed to traceCall (%d)", pid);
    return false;
}
```



这样我们就能够在目标进程中调用特定的方法了，同理适用于`dlopen`，这就不再赘述，下面主要讲so被加载之后，是如何Hook来替换函数。



# ElfHook
在对ELF文件格式有了大致的认识之后，我们就可以基于ELF文件的执行视图，来进行ElfHook，整体思路也是参考linker加载so库的流程，如果对于linker加载so库的流程比较熟悉的话那对ElfHook应该也不陌生。
> 分析是基于32位的ELF文件格式，代码中兼容了64位。

## 整体流程
先来快速的过一下Hook流程：
1.读取ELF文件头，获取程序头表位置等信息
2.遍历程序头表，找到类型为PT_DYNAMIC的段区，动态链接段
3.再遍历动态链接段区，获得不同类型的节区，初始化相关参数
4.根据传入的符号值`symbol`，经过哈希计算得出该符号在`符号表(.dynsym)`的索引值，根据索引值获取`符号表`的`st_name`(非0即为该符号在`字符串表(.dynstr)`中的索引即`symidx`)
5.遍历重定位表`pltrel`和`rel`，计算每个表项在`字符串表`的索引值等于`symidx`的重定位表项，并从该表项中得到对应的地址`addr`(这个地址里存放的 是`symbol`函数的实际地址)
6.获取`addr`所在页的起始地址，修改改页的读写权限
7.取出`addr`中`symbol`实际地址，保存到`old_addr`备用，将我们自己写的函数的地址`new_addr`写入`addr`中
8.系统调用，清除缓存，保证替换生效

基本流程如上，就是照着ELF文件的格式来解析，规则很死，下面主要讲之前做错的和报错比较多的地方。

## 哈希查找
通过计算`symbol`的哈希值，查找哈希表得到索引值`symidx`。
目前ELF有2种Hash表，一种就是DT_HASH，详见`ELF文件格式系统3.8.6`，另一种就是DT_GNU_HASH，详见参考。
如果哈希表的类型是DT_GNU_HASH时，是存在`symbol`通过GNU_HASH无法找到对应的`symidx`的，根据文档原文解释，`symndx`是能被GNU_HASH找到的第一个符号的索引值。假设符号表的项数为`dynsymcount`，那么可以被找到的项数为`dynsymcount-symndx`。所以当通过GNU_HASH无法找到的时候，就要手动遍历符号表的前`symndx`项来计算`symidx`，部分代码如下：
```c
		if (0 == gnuLookup(symbol, sym, symidx)) {
            return 0;
        }

        ElfW(Sym) *curr;
        for(uint32_t i=0; i<this->gnuSymndx; i++) {
            curr = this->symTable + i;
            if (0 == strcmp(this->strTable+curr->st_name, symbol)) {
                *symidx = i;
                *sym = curr;
                return 0;
            }
        }
        LOGE("not found %s in %s before gnu symbol index %d", symbol, this->moduleName, this->gnuSymndx);
        return -1;
```


## 基址偏移计算
先来看看6.0源码中[Linker](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker_phdr.cpp#307)加载so库的时候，ElfReader为so库分配内存空间的部分代码:
```c
307  bool ElfReader::ReserveAddressSpace(const android_dlextinfo* extinfo) {
308    ElfW(Addr) min_vaddr;
309    load_size_ = phdr_table_get_load_size(phdr_table_, phdr_num_, &min_vaddr);
  		...
315    uint8_t* addr = reinterpret_cast<uint8_t*>(min_vaddr);
316    void* start;
  		...
341    int mmap_flags = MAP_PRIVATE | MAP_ANONYMOUS;
342    start = mmap(mmap_hint, load_size_, PROT_NONE, mmap_flags, -1, 0);
  		...
351    load_start_ = start;
352    load_bias_ = reinterpret_cast<uint8_t*>(start) - addr;
353    return true;
354  }
```

这段代码先是计算了so库需要的空间，然后调用系统函数mmap来映射。

> 值得注意的是`min_vaddr`，通常情况下SO是不会指定加载的基址，即min_vaddr=0。但是如果SO指定了加载基址，并且地址不是页对齐的，就会导致实际映射地址与指定加载地址有偏差，这个偏差值就是`load_bias_`，所以在针对虚拟地址计算的时候需要使用`load_bias_`来修正。



而Linker也有计算`load_bias`地址的方法，代码中计算值也是跟这个一样的原理：

```
3329/* Compute the load-bias of an existing executable. This shall only
3330 * be used to compute the load bias of an executable or shared library
3331 * that was loaded by the kernel itself.
3332 *
3333 * Input:
3334 *    elf    -> address of ELF header, assumed to be at the start of the file.
3335 * Return:
3336 *    load bias, i.e. add the value of any p_vaddr in the file to get
3337 *    the corresponding address in memory.
3338 */
3339  static ElfW(Addr) get_elf_exec_load_bias(const ElfW(Ehdr)* elf) {
3340    ElfW(Addr) offset = elf->e_phoff;
3341    const ElfW(Phdr)* phdr_table =
3342    reinterpret_cast<const ElfW(Phdr)*>(reinterpret_cast<uintptr_t>(elf) + offset);
3343    const ElfW(Phdr)* phdr_end = phdr_table + elf->e_phnum;
3344
3345    for (const ElfW(Phdr)* phdr = phdr_table; phdr < phdr_end; phdr++) {
3346      if (phdr->p_type == PT_LOAD) {
3347        return reinterpret_cast<ElfW(Addr)>(elf) + phdr->p_offset - phdr->p_vaddr;
3348      }
3349    }
3350    return 0;
3351  }
```













# 参考
[Ptrace](https://en.wikipedia.org/wiki/Ptrace)
[ Android Hook学习之ptrace函数的使用](http://blog.csdn.net/qq1084283172/article/details/45967633)
[ptrace函数详解  ](http://blog.163.com/yuanxiaohei@126/blog/static/6742308720123683433989/)
[arm 寄存器](https://www.cnblogs.com/xiaojinma/archive/2012/12/19/2825481.html)
[waitpid之status意义解析  ](http://tsecer.blog.163.com/blog/static/15018172012323975152/)

[gnu_hash](https://blogs.oracle.com/ali/gnu-hash-elf-sections)
[Linker和So加壳](http://dev.qq.com/topic/57e3a3bc42eb88da6d4be143)

[Android的so注入( inject)和函数Hook(基于got表)](http://blog.csdn.net/QQ1084283172/article/details/53942648?locationNum=1&fps=1)
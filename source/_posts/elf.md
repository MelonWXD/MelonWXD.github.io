---
title: ELF格式分析
date: 2017-11-19 15:35:45
tags: [注入] 
categories: Android逆向安全  
---
Linux下的ELF(可执行可链接)格式的分析
<!-- more -->
# 目标文件 Object File
一个简单的c文件，经过预处理-编译-汇编-链接，四个耳熟能详的步骤后，就变成了一个可执行的目标文件。（如果根据Object File直译的话应该是对象文件，但是CSAPP写的是目标文件，还是听大佬的吧）   
目标文件有三种形式：
- 可重定位文件（Relocatable file）：这是由汇编器汇编生成的.o文件，链接器会使用一个或多个的可重定位文件作为输入，生成一个可执行文件
- 可执行文件（Executable file）：可以直接复制到内存中执行的文件
- 可共享文件（Shared Object file）：也是Android中常说的so库。所谓的动态库文件，在运行时被动态的加载进内存，由动态链接器来负责链接so库和可执行文件的执行。

这些目标文件都是按照特定的文件格式来组织的，比如Windows下是PE，Mac下是Mach-O，Linux/unix下就是ELF格式。  
这里主要就是以Linux下的so的文件格式ELF来做分析，因为在逆向APK中主要也是针对so库做注入。
# EFL文件格式
先放2张图，可以直观的了解ELF文件格式
![](http://blog.chinaunix.net/photo/94212_101201164532.jpg)
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1511088380577&di=7d77512485a60706d6fd1ab9f949f7af&imgtype=0&src=http%3A%2F%2Fwww.yeolar.com%2Fmedia%2Fnote%2F2012%2F03%2F20%2Flinux-linking%2Ffig2.png)

## 结构综述
- ELF文件头 ： 对ELF文件整体的一个描述
- 程序头部表 Program Header Table ：描述下面的段区Segment，告诉系统如何创建进程影像
- 节区Section/段区Segment ：section从链接角度具体描述这个elf文件，segment是从运行角度来描述这个elf文件
- 节区头部表 Section Header Table ：描述上面的各个节区Section的属性信息


## 链接视图和执行视图
为什么要区分链接的视图和执行视图，ok，我不知道
> 当ELF文件被加载到内存中后，系统会将多个具有相同权限（flg值）section合并一个segment。操作系统往往以页为基本单位来管理内存分配，一般页的大小为4096B，即4KB的大小。同时，内存的权限管理的粒度也是以页为单位，页内的内存是具有同样的权限等属性，并且操作系统对内存的管理往往追求高效和高利用率这样的目标。ELF文件在被映射时，是以系统的页长度为单位的，那么每个section在映射时的长度都是系统页长度的整数倍，如果section的长度不是其整数倍，则导致多余部分也将占用一个页。而我们从上面的例子中知道，一个ELF文件具有很多的section，那么会导致内存浪费严重。这样可以减少页面内部的碎片，节省了空间，显著提高内存利用率。

# 分析
下面我就从链接的角度，结合linux的命令`readelf`来分析Android的GUI系统中使用的libsurfaceflinger.so(Android 4.4)文件来逐一介绍EFL文件 节区头部表 与一些与重定位相关的节区。

## ELF文件头
ELF文件头对应的结构体Elf32_Ehdr如下：
```
#define EI_NIDENT 16
typedef struct {
       unsigned char e_ident[EI_NIDENT];
       ELF32_Half e_type;
       ELF32_Half e_machine;
       ELF32_Word e_version;
       ELF32__Addr e_entry;
       ELF32_Off e_phoff;
       ELF32_Off e_shoff;
       ELF32_Word e_flags;
       ELF32_Half e_ehsize;
       ELF32_Half e_phentsize;
       ELF32_Half e_phnum;
       ELF32_Half e_shentsize;
       ELF32_Half e_shnum;
       ELF32_Half e_shstrndx;
}Elf32_Ehdr;
```
使用`readelf -h libsurfaceflinger.so `可以让系统来自动解析文件头hex值对应的实际意义
```
duoyi@duoyi-OptiPlex-7010:~/Desktop/todo$ readelf -h libsurfaceflinger.so 
ELF 头：
  Magic：   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF32
  数据:                              2 补码，小端序 (little endian)
  版本:                              1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              DYN (共享目标文件)
  系统架构:                          ARM
  版本:                              0x1
  入口点地址：               0x0
  程序头起点：          52 (bytes into file)
  节区头起点:          201044 (bytes into file)
  标志：             0x5000000, Version5 EABI
  本头的大小：       52 (字节)
  程序头大小：       32 (字节)
  Number of program headers:         8
  节头大小：         40 (字节)
  节头数量：         24
  字符串表索引节头： 23

```

## 节区头部表
同样的，节区头部表的表项对应的结构体Elf32_Shdr如下：
```
 typedef struct elf32_shdr { 
          Elf32_Word    sh_name;  
          Elf32_Word    sh_type; 
          Elf32_Word    sh_flags; 
          Elf32_Addr    sh_addr;    //在内存中的虚地址
          Elf32_Off     sh_offset;   //对应的Section在文件中的偏移
          Elf32_Word    sh_size; 
          Elf32_Word    sh_link; 
          Elf32_Word    sh_info; 
          Elf32_Word    sh_addralign; 
          Elf32_Word    sh_entsize; 
        } Elf32_Shdr; 
```
可能说的有点绕，节区头部表本身就含有多个项，每个项对应着上面说的Section，一个是表，一个是项，一个节区头表，含有多个节区项的描述结构体，该结构体就是Elf32_Shdr。那么这个节区头表的位置我们是如何确定的呢，这是在ELF文件头中确定的，往上看可以找到
```
  节区头起点:          201044 (bytes into file)
```
这个是起始位置，结束位置一般就是文件尾巴了。201044对应的16进制值为0x31154，我们使用命令`readelf -S libsurfaceflinger.so `来解析节区头看看：
```
duoyi@duoyi-OptiPlex-7010:~/Desktop/todo$ readelf -S libsurfaceflinger.so 
共有 24 个节头，从偏移量 0x31154 开始：

节头：
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        00000134 000134 000013 00   A  0   0  1
  [ 2] .dynsym           DYNSYM          00000148 000148 0020a0 10   A  3   1  4
  [ 3] .dynstr           STRTAB          000021e8 0021e8 00454c 00   A  0   0  1
  [ 4] .hash             HASH            00006734 006734 001054 04   A  2   0  4
  [ 5] .rel.dyn          REL             00007788 007788 0069f0 08   A  2   0  4
  [ 6] .rel.plt          REL             0000e178 00e178 000f08 08   A  2   7  4
  [ 7] .plt              PROGBITS        0000f080 00f080 0016a0 00  AX  0   0  4
  [ 8] .text             PROGBITS        00010720 010720 015828 00  AX  0   0  8
  [ 9] .ARM.exidx        ARM_EXIDX       00025f48 025f48 001958 08  AL  8   0  4
  [10] .ARM.extab        PROGBITS        000278a0 0278a0 000390 00   A  0   0  4
  [11] .rodata           PROGBITS        00027c30 027c30 002ca8 00   A  0   0  4
  [12] .data.rel.ro.loca PROGBITS        0002c7d8 02b7d8 000108 00  WA  0   0  8
  [13] .fini_array       FINI_ARRAY      0002c8e0 02b8e0 000004 00  WA  0   0  4
  [14] .init_array       INIT_ARRAY      0002c8e4 02b8e4 000018 00  WA  0   0  4
  [15] .data.rel.ro      PROGBITS        0002c900 02b900 004d80 00  WA  0   0  8
  [16] .dynamic          DYNAMIC         00031680 030680 000150 08  WA  3   0  4
  [17] .got              PROGBITS        000317d4 0307d4 00082c 00  WA  0   0  4
  [18] .data             PROGBITS        00032000 031000 000009 00  WA  0   0  4
  [19] .bss              NOBITS          0003200c 031009 000098 00  WA  0   0  4
  [20] .comment          PROGBITS        00000000 031009 000010 01  MS  0   0  1
  [21] .note.gnu.gold-ve NOTE            00000000 03101c 00001c 00      0   0  4
  [22] .ARM.attributes   ARM_ATTRIBUTES  00000000 031038 00003a 00      0   0  1
  [23] .shstrtab         STRTAB          00000000 031072 0000e0 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

```
打印出的第一行信息
>共有 24 个节头，从偏移量 0x31154 开始：  

和我们根据ELF文件头算出来的一样。  
可以看到下面输出了很多个表项，每个表项的结构就是上述的Elf32_Shdr了，下面挑几个相关的来讲。  
### 符号表 .dynsym

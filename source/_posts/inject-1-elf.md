---
title: 注入之道(一) ELF格式分析
date: 2017-11-19 15:35:45
tags: [注入] 
categories: Android逆向安全  
---
32位ELF(可执行可链接文件)格式的分析
<!-- more -->
# 目标文件 Object File
一个简单的c文件，经过预处理-编译-汇编-链接，四个耳熟能详的步骤后，就变成了一个可执行的目标文件。  
目标文件有三种形式：
- 可重定位文件（Relocatable file）：这是由汇编器汇编生成的.o文件，链接器会使用一个或多个的可重定位文件作为输入，生成一个可执行文件
- 可执行文件（Executable file）：可以直接复制到内存中执行的文件
- 可共享文件（Shared Object file）：也是Android中常说的so库。所谓的动态库文件，在运行时被动态的加载进内存，由动态链接器来负责链接so库和可执行文件的执行。

这些目标文件都是按照特定的文件格式来组织的，比如Windows下是PE，Mac下是Mach-O，Linux/unix下就是ELF格式。  
这里主要就是以Linux下的so文件格式ELF来做分析，因为在逆向APK中主要也是针对so库做注入。
# EFL文件格式
先放2张图，可以直观的了解ELF文件格式
![](http://blog.chinaunix.net/photo/94212_101201164532.jpg)
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1511088380577&di=7d77512485a60706d6fd1ab9f949f7af&imgtype=0&src=http%3A%2F%2Fwww.yeolar.com%2Fmedia%2Fnote%2F2012%2F03%2F20%2Flinux-linking%2Ffig2.png)

## 结构综述
- ELF文件头 ： 对ELF文件整体结构的一个描述
- 程序头部表 Program Header Table ：描述下面的段区Segment，告诉系统如何创建进程影像
- 节区Section ：section从链接角度具体描述这个elf文件
- 段区Segment ：segment是从运行角度来描述这个elf文件，segment包含多个section
- 节区头部表 Section Header Table ：描述上面的各个节区Section的属性信息




**为什么要区别节区和段区？**

> 当ELF文件被加载到内存中后，系统会将多个具有相同权限（flag值）section合并一个segment。操作系统往往以页为基本单位来管理内存分配，一般页的大小为4096B，即4KB的大小。同时，内存的权限管理的粒度也是以页为单位，页内的内存是具有同样的权限等属性，并且操作系统对内存的管理往往追求高效和高利用率这样的目标。ELF文件在被映射时，是以系统的页长度为单位的，那么每个section在映射时的长度都是系统页长度的整数倍，如果section的长度不是其整数倍，则导致多余部分也将占用一个页。而我们从上面的例子中知道，一个ELF文件具有很多的section，那么会导致内存浪费严重。这样可以减少页面内部的碎片，节省了空间，显著提高内存利用率。



# 分析
下面我就结合linux的命令`readelf`来分析Android的GUI系统中使用的`/system/lib/libsurfaceflinger.so`文件来逐一介绍EFL文件 节区头部表 与一些与重定位相关的节区。

> Android 4.4 32位所以结构体都是ELF32开头，64位的文件格式即为ELF64 

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
  程序头表项数:         8
  节头大小：         40 (字节)
  节头数量：         24
  字符串表索引节头： 23

```

## “节”头表
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
这个是起始位置，结束位置一般就是文件末尾。201044对应的16进制值为0x31154，我们使用命令`readelf -S libsurfaceflinger.so `来解析节区头看看：
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
可以看到打印出来的第一行信息和我们根据ELF文件头算出来的一样。  
可以看到输出了很多个表项，每个表项的结构就是上述的Elf32_Shdr了，下面挑几个相关的来讲。  
### 符号表 .dynsym
符号表包含用来定位、重定位程序中符号定义和引用的信息，简单的理解就是符号表记录了该文件中的所有符号。所谓的符号就是经过修饰了的函数名或者变量名，不同的编译器有不同的修饰规则。例如符号_ZL15global_static_a，就是由global_static_a变量名经过修饰而来。  
通过命令`readelf -s libsurfaceflinger.so `来查看.dynsym的Section内容
```
duoyi@duoyi-OptiPlex-7010:~/Desktop/todo$ readelf -s libsurfaceflinger.so 

Symbol table '.dynsym' contains 522 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_finalize
     2: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_atexit
     3: 00000000     0 FUNC    GLOBAL DEFAULT  UND __aeabi_unwind_cpp_pr0
     4: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNK7android22ISurfaceCom
     5: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZN7android10VectorImpl13
     6: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZN7android16SortedVector
	 ...
     520: 00032009     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start
     521: 000320a4     0 NOTYPE  GLOBAL DEFAULT  ABS _end

```
也可以通过`readelf -p [sectionIdx] libsurfaceflinger.so `来以string的方式查看指定序号的Section内容，值没有经过格式化，比如这里的.dynsym序号是2，那么`readelf -p 2 libsurfaceflinger.so`输出的内容为：
```
duoyi@duoyi-OptiPlex-7010:~/Desktop/todo$ readelf -p 2 libsurfaceflinger.so 

String dump of section '.dynsym':
  [    40]  4
  [    50]  s
  [    b0]  .^A
  [    c0]  \^A
  [    d0]  c^A
  [   120]  A^B
  [   130]  G^B
  [   140]  _^B
  [   1b0]  2^C
  [   1c0]  Z^C
  ...
```

跟上面分析节区表和节区表的表项一样，符号表的结构是节区表的表项，那符号表的表项结构体是怎么表示的呢，如下
```
typedef struct {  
     Elf32_Word st_name;      //符号表 表项的名称。
     //如果该值非0，则表示符号名在字符串表(.dynstr)中的索引值(offset)，为0即符号表项没有名称。
     Elf32_Addr st_value;       //符号的取值。依赖于具体的上下文，可能是一个绝对值、一个地址等等。
     Elf32_Word st_size;         //符号的尺寸大小。例如一个数据对象的大小是对象中包含的字节数。
     unsigned char st_info;    //符号的类型和绑定属性。
     unsigned char st_other;   
     Elf32_Half st_shndx;        //每个符号表项都以和其他节区的关系的方式给出定义。
　　　　　　　　　　　　　//此成员给出相关的节区头部表索引。
} Elf32_Sym; 
```
### 字符串表 .dynstr
根据上面说的 符号表结构体的st_name项，非0值表示该符号名在字符串表中的索引，那么自然而然的就要介绍一下字符串表了。  
字符串表中包含若干以 null 结尾的字符串，这些字符串通常是 symbol 或 section 的名字。当 ELF 文件的其它部分需要引用字符串时，只需提供该字符串在字符串表中的位置索引即可。  

下图为一个长度为25字节的字符串表示例：  
![](http://images0.cnblogs.com/blog2015/763648/201506/302242024621822.png)
字符串引用示例：  
![](http://images0.cnblogs.com/blog2015/763648/201506/302242336658657.png)  

根据上面ELF文件头 字符串表的索引为3，通过`readelf -p 3 libsurfaceflinger.so`来打印一下：
```
duoyi@duoyi-OptiPlex-7010:~/Desktop/todo$ readelf -p 3 libsurfaceflinger.so 

String dump of section '.dynstr':
  [     1]  __cxa_finalize
  [    10]  __cxa_atexit
  [    1d]  __aeabi_unwind_cpp_pr0
  ..
  [  451a]  libc.so
  [  4522]  libstdc++.so
  [  452f]  libm.so
  [  4537]  libsurfaceflinger.so
```

### 重定位表 
在编译-汇编这2个操作中，编译器和汇编器为每个文件创建程序地址一般都是从0开始，但是实际加载时模块的基址肯定不是0，为了避免加载地址重叠，就引入了重定位这个东西。重定位就是为程序不同部分分配加载地址，调整程序中的数据和代码以反映所分配地址的过程。简单的言之，则是将程序中的各个部分映射到合理的地址上来。

**.rel.text .rel.dyn .rel.plt .plt  .got .got.plt的关系**

- .rel.text：定位的地方在.text段内，以offset指定具体要定位位置。在链接时候由链接器完成。

- .rel.dyn：重定位的地方在.got段内，主要是针对外部数据变量符号。不支持延迟重定位(Lazy)，通常是在so文件执行时就在.init段中进行重定位操作。

- .rel.plt：重定位的地方在.got.plt段内, 主要是针对外部函数符号。一般是函数首次被调用时候重定位。

- .plt：Procedure Linkage Table，过程链接表。所有对外部函数的调用都经过PLT再到GOT的一个调用过程。

- .got：Global Offset Table，全局偏移表，存放着调用外部函数的实际地址（第一次存放的是PLT中的指令，PLT执行完之后会把计算得到的实际值再存到GOT中）。

- .got.plt：ELF将GOT拆分成两个表 .got和.got.plt,前者用来保存全局变量引用的地址，后者用来保存函数引用的地址。

  > Android ARM 下需要处理两个重定位表，plt_rel 和 rel，plt 指的是延迟绑定，但是 Android 目前并不对延迟绑定做特殊处理，直接与普通的重定位同时处理。

### 重定位

[【ARM】安卓SO中GOT REL PLT 作用与关系](https://bbs.pediy.com/thread-221821.htm)







## “程序”头表

上面分析了这么多的Section区，但实际上so库加载到内存中，是按照执行视图来查找各个重定位表的，所以下面就来分析一下执行视图中必须存在的程序头表（Program Header Table）。  
程序头表表项的结构体Elf32_phdr如下：
```
typedef struct {  
    Elf32_Word p_type;           //此数组元素描述的段的类型，或者如何解释此数组元素的信息。 
    Elf32_Off  p_offset;           //此成员给出从文件头到该段第一个字节的偏移
    Elf32_Addr p_vaddr;         //此成员给出段的第一个字节将被放到内存中的虚拟地址
    Elf32_Addr p_paddr;        //此成员仅用于与物理地址相关的系统中。system V忽略所有应用程序的物理地址信息。
    Elf32_Word p_filesz;         //此成员给出段在文件映像中所占的字节数。可以为0。
    Elf32_Word p_memsz;     //此成员给出段在内存映像中占用的字节数。可以为0。
    Elf32_Word p_flags;         //此成员给出与段相关的标志。
    Elf32_Word p_align;        //此成员给出段在文件中和内存中如何对齐。
} Elf32_phdr;
```
这里的p_type就代表了当前Segment的类型，重点介绍2个：
- PT_LOAD：指定可装入段，通过 p_filesz 和 p_memsz 进行描述。只有类型为PT_LOAD的段才是需要装入的。当然在装入之前，需要确定装入的地址，只要考虑的就是页面对齐，还有该段的p_vaddr域的值。确定了装入地址后，就通过elf_map()建立用户空间虚拟地址空间与目标映像文件中某个连续区间之间的映射，其返回值就是实际映射的起始地址。
- PT_DYNAMIC：动态链接相关，如果目标文件参与动态链接，则其程序头表将包含一个类型为 PT_DYNAMIC 的元素。此段包含 .dynamic 节。

### 动态节.dynamic
和我们重定位相关的动态节的结构体如下：
```
typedef struct {
        Elf32_Sword d_tag;
        union {
                Elf32_Word      d_val;
                Elf32_Addr      d_ptr;
        } d_un;
} Elf32_Dyn;
```
- d_tag：表明该动态节类型，有DT_PLTGOT、DT_HASH、DT_STRTAB、DT_SYMTAB、DT_RELA等等
- d_un->d_val：根据tag表示的不同有不同的意思，多数情况下表示该节大小值
- d_un->d_ptr：表示该节的虚拟地址（文件的虚拟地址与实际执行过程中的虚拟地址不一定匹配）

## ELF文件格式图例
![](http://img.blog.csdn.net/20140918103240781?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VpZ3V6aTExMTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
# 参考
[【ARM】安卓SO中GOT REL PLT 作用与关系](https://bbs.pediy.com/thread-221821.htm)

[EFL PLT GOT跳转关系](http://blog.csdn.net/lifeshow/article/details/29597401)
[ELF文件中的.plt .rel.dyn .rel.plt .got .got.plt的关系](https://www.cnblogs.com/leo0000/p/5604132.html)  
[ELF文件格式解析](http://www.07net01.com/2016/05/1534423.html)  
[difference-between-got-and-got-plt](https://stackoverflow.com/questions/11676472/what-is-the-difference-between-got-and-got-plt-section#comment16231745_11676472)
---
title: JVM虚拟机，Dalvik虚拟机，ART虚拟机分析
date: 2017-10-10 11:40:24
tags: [jvm,Dalvik,ART]
categories: Dalvik
---
JVM虚拟机，Dalvik虚拟机，ART虚拟机分析
<!-- more -->
## .java .class .dex .apk
- .java 源文件
- .class 字节码文件 可以被jvm执行
- .dex Dalvik支持的字节码文件 可以被dalvik执行
- .apk 资源文件和dex文件打包后，再经过签名、对齐等操作变成apk文件（Android package）

### apk安装流程
![](https://cloud.githubusercontent.com/assets/21374839/26571436/8779c70c-4548-11e7-9e9f-7ff265b32837.png)
  
  
## Dalvik、JIT 与 ART、AOT
### Dalvik
通俗的讲，Dalvik虚拟机就是Android平台上的JVM虚拟机。主要对JVM的修改如下：
1. 将原来 class 文件进行优化，例如将其中的常量冗余信息进行合并，提高虚拟机解析效率
2. 修改 JVM 运行时基于栈的数据结构修改为 Dalvik 基于寄存器的数据结构，数据访问方式更快，运行效率更高  

所以才会有了.dex文件格式，Dalvik将.apk文件解压（即安装apk的过程），转class.dex转换成ODEX文件（转换过程还会根据当前系统进行优化）存储在本地，运行APP的时候就是找到这个ODEX文件并通过Dalvik来执行。
### JIT
JAVA是解释型语言，JVM执行字节码的时候通常有2个方案：
1. 在执行前一次性把所有指令都编译成本地代码，然后再执行
2. 逐条解释并执行  

JIT是Just In Time，即时编译。在JVM解释执行字节码的时候，多次调用的程序段才被编译，编译后存放在内存中，下次直接执行编译后的本地代码。至于为什么采用这种折中的优化方法，是因为在运行时，将字节码翻译成机器码也是需要时间的，如果大部分代码的执行次数很少，那么编译花费的时间不一定少于执行的时间。

就HotSpot虚拟机来说的话有client和server两种模式，又有一定程度的[不同](http://ifeve.com/hotspot-jit/)

### 使用AOT策略的ART虚拟机
JIT是运行时编译，这样可以对执行次数频繁的dex代码进行编译和优化，虽然可以加快Dalvik运行速度，但是还是有弊病，那就是将dex翻译为本地机器码也要占用时间，所以Google在4.4之后推出了ART，用来替换Dalvik（在5.0中彻底替换）。  
ART的策略与Dalvik不同，在ART 环境中，应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用。之后打开App的时候，不需要额外的翻译工作，直接使用本地机器码运行，因此运行速度提高。这就是AOT，Ahead Of Time ，相应的会消耗掉更多的存储空间以及导致安装时间加长。
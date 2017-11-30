---
title: JVM Specification Chapter2    
date: 2017-09-24 20:18:29  
tags: [JVM Specification,翻译]  
categories: JVM
---
Oracle JVM规范 简单翻译/笔记 - Chapter2
<!-- more -->
## 原文地址
[原文地址](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html)  
## Chapter2：JVM的结构
## class文件格式
[等老子坚持第四章了再说吧](www.fuckyou.com)
## 数据类型 
与Java语言类似，JVM能够处理的2种类型为 原始类型和引用类型。相应的，这两种类型也可以被保存为比变量、作为参数传递、方法的返回值和运算操作。
JVM希望几乎所有的类型检查在运行时都已经完成，通常是由IDE来做这些检查的。原始类型的值无需被硬件标记，不需要在runtime去决定他们的类型，也不需要和引用类型的值区分开来，因为操作这些原始类型的数据的指令集（ instruction set ）本身就已经指出了那些被操作的原始数据的类型。煮个例子，iadd，ladd，fadd，dadd都是JVM相加2个数的指令，但是每个指令都分别对应了int，long，float，double这些数据类型。
[jvm指令集](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.1)  
JVM明确支持对象（objects），一个对象不是动态分配的类实例就是一个数组。一个对象的引用，在JVM中就叫做引用类型。引用类型的值可以被认为是指向对象的指针。一个对象可以有多个引用，对象的操作总是通过引用类型的值来传递和测试。  
### returnAddress 类型和值
returnAddress通常被JVM的jsr, ret, 以及jsr_w指令调用。returnAddress类型的值指向一条JVM指令的操作码。该类型在Java语言中不存在与之对应的类，无法在运行时修改。
## 运行时数据区(Run-Time Data Areas)
在执行程序的时候，JVM定义了各种运行时数据区。一些区域跟JVM具有相同的生命周期(JVM进程)，其他的数据是跟每个线程具有相同的生命周期。


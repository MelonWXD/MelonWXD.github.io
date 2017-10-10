---
title: Kotlin 设计模式—适配器模式  
date: 2017-10-10 22:32:50
tags: [设计模式]  
categories: Kotlin
---
Kotlin设计模式之适配器模式
<!-- more -->  
## 适配器模式
## 简介
适配器模式：将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。  
在我看到这句话的时候，我的心里想的是：为什么你要让两个接口不兼容的东西一起工作？可惜kotlin学的不是很深，后续场景应该可以结合ListView的Adapter来讲讲，虽然我觉得两者没什么关系。。  

### 适用场景
1. 业务的接口与工作的类不兼容，（比如：类中缺少实现接口的某些方法）但又需要两者一起工作
2. 在现有接口和类的基础上为新的业务需求提供接口

## 实例
生活中，我们常用的插头是二插头和三插头的，那么在开发的时候，通常我们的业务接口可以这样写：
```kotlin
interface TwoPlug{
    fun powerWithTwo()
}
interface ThreePlug{
    fun powerWithThree()
}
class TwoPlugImp :TwoPlug{
    override fun powerWithTwo() {
        println("正在使用二插头充电")
    }
}
class ThreePlugImp :ThreePlug{
    override fun powerWithThree() {
        println("正在使用三插头充电")
    }
}

```
ok，简直完美，要啥有啥，但是天不遂人愿，你的客户只有三插头的充电器（业务逻辑的接口方法），但是他家的插座都是二插头的（实际编码的类），于是你要弄个适配器使得三插头的充电器可以插在二插头的插座上工作。

```kotlin

class Two2ThreeAdapter : TwoPlug {
    var threePlug = ThreePlugImp()
    override fun powerWithTwo() {
        threePlug.powerWithThree()
    }
}

fun main(args: Array<String>) {
    val plug = Two2ThreeAdapter()
    plug.powerWithTwo()
}
```
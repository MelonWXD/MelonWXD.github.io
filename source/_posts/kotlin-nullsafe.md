---
title: Kotlin空指针安全  
date: 2017-09-29 17:03:21    
tags:  [Kotlin语法]
categories:  Kotlin
---
Kotlin Null Safety 学习笔记
<!-- more -->

 
## ? 可空 非空 
```kotlin
    var a: String? = null  //type is String? 可以赋null值
    var b: String = "b" //type is String
    b=a //报错 类型不匹配 
```
## 空值检查
```kotlin
    var b: String? = null //
    val l = if(b!=null)b.length else -1
    //or..
    if (b != null && b.length > 0) {
        print("String of length ${b.length}")
    } else {
        print("Empty string")
    }
```
## 安全调用：?. 和 let
如果每次都对可空的变量做空值判断，那不是和java一样啰嗦，所以kotlin还有一种简洁的调用方法
```
b?.length // return null if b==null
bob?.department?.head?.name //链式调用
``` aInt: Int? = a as? Int
如果只想返回非空值，可以使用let关键字来忽略空值
```kotlin
val listWithNulls: List<String?> = listOf("A","B", null)
for (item in listWithNulls) {
     item?.let { println(it) } // prints A and B
}
```

## ?: 操作符 
 如果 ?: 左边表达式不为空则返回，否则返回右边的表达式
```
val l: Int = if (b != null) b.length() else -1
```
等同于
```
val l = b.length()?: -1
```

## !! 操作符
返回一个非空的 b 或者抛出一个 b 为空的 异常
```
val l = b !!.length()
```

## 安全转换
一般我们在做类型转换的时候，如果被转换的不是我们预估的类型，会报一个ClassCastException异常，这里可以使用安全转换来做，当失败的时候返回一个null
```
val aInt: Int? = a as? Int
```
## null集合
如果想要过滤一个集合中的空值，可以使用
```
val nullableList: List<Int?> = listOf(1, 2, null, 4)
val intList: List<Int> = nullableList.filterNotNull()
```
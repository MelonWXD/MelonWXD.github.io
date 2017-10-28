---
title: kotlin委托机制
date: 2017-10-25 20:27:19
tags:  [Kotlin语法,设计模式]
categories:  Kotlin
---
Kotlin中的委托机制
<!-- more -->

## 类委托
Kotlin通过`by`关键字可以简单的实现委托模式。  
委托模式，用java的话来说，就是通过组合，让类A持有其他类B的实例，然后类A中的部分方法实际上都是通过调用的类B这个实例的方法来实现的。

### kotlin实现委托模式
Activity中的onClick方法，实际上是委托给OnClickListener的实例来实现的，简单易懂。
```kotlin
interface Listener {
    fun onClick()
}

class OnClickListener : Listener {
    override fun onClick() {
        print("OnClickListener")
    }
}

class Activity : Listener {
    val mListener = OnClickListener()
    override fun onClick() {
        mListener.onClick()
    }
}

fun main(args: Array<String>) {
    var act = Activity()
    act.onClick()
}
```
### by关键字简洁地实现委托模式
上面的实现，乍一看跟java没两样，可kotlin要的就是简洁、优雅，要是还跟java一样罗里吧嗦，怎么体现这门语言的逼格？所以就引入了 `by` 关键字来实现委托模式。  
原理都差不多，在类Activity的构造方法中传入OnClickListener实例即可，直接上代码吧。
```kotlin
interface Listener {
    fun onClick()
}

class OnClickListener : Listener {
    override fun onClick() {
        print("OnClickListener")
    }
}

class Activity(listener: OnClickListener) : Listener by listener

fun main(args: Array<String>) {
    var listener = OnClickListener()
    var act = Activity(listener)
    act.onClick()
}
```


## 代理属性
理解有误，修改中。。

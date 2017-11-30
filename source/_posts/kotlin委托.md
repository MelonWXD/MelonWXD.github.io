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
Kotlin也支持代理属性，语法定义如下：  
`val/var <property name>: <Type> by <expression>`  
- var/val：属性类型(可变/只读)
- name：属性名称
- Type：属性的数据类型
- expression：代理类

```kotlin
class Example {
    var p: String by Delegate()
}

class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name} in $thisRef.'")
    }
}

fun main(args: Array<String>) {
    val e = Example()
    println(e.p)
    e.p = "NEW"
}
```
输出
```
Example@25f38edc, thank you for delegating 'p' to me!
NEW has been assigned to 'p in Example@25f38edc.'
```

Kotlin 标准库为几种常用的代理提供了工厂方法:
- 延迟加载属性(lazy property): 属性值只在初次访问时才会计算,然后保存结果，后面直接返回这个结果
- 可观察属性(observable property): 属性发生变化时, 可以向监听器发送通知
- 将多个属性保存在一个 map 内, 而不是保存在多个独立的域内


### 延迟加载的属性
lazy() 是一个接受 lamdba 并返回一个实现延迟属性的代理：第一次调用 get() 执行 lamdba 并传递 lazy() 并存储结果，以后每次调用 get() 时只是简单返回之前存储的值。
> 注意: var类型属性不能设置为延迟加载属性，因为在lazy中并没有setValue(…)方法  
lazy操作符是线程安全的。如果在不考虑多线程问题或者想提高更多的性能，也可以使用lazy(LazyThreadSafeMode.NONE){ … } 


```kotlin
fun lazyinit(): Int {
    println("lazy is called")
    return 1
}

val age by lazy {
    lazyinit()
}

fun main(args: Array<String>) {
    println("age: $age") 
    println("age: $age") 

}
```
输出
```
lazy is called
age: 1
age: 1
```

### 可观察的属性
Delegates.observable() 需要两个参数：一个初始值和一个用于修改的 handler  
handler有三个参数：一个将被赋值的属性，旧值，新值。
```kotlin
var name: String by Delegates.observable("initName", {
    kProperty, oldName, newName ->
    println("kProperty：${kProperty.name} | oldName:$oldName | newName:$newName")
})


fun main(args: Array<String>) {

    println("name: $name")
    name = "newName1"
    name = "newName2"
    println("name: $name")
}
```
输出
```
name: initName
kProperty：name | oldName:initName | newName:newName1
kProperty：name | oldName:newName1 | newName:newName2
name: newName2
```

Delegates.vetoable() 也需要两个参数：一个初始值和一个用来在保存新值之前做一些条件判断，来决定是否将新值保存。
修改上述部分代码：
```kotlin
var name: String by Delegates.vetoable("initName", {
    kProperty, oldName, newName ->
    println("kProperty：${kProperty.name} | oldName:$oldName | newName:$newName")
    newName.contains("1")
})
```
输出
```
name: initName
kProperty：name | oldName:initName | newName:newName1
kProperty：name | oldName:newName1 | newName:newName2
name: newName1
```

### 在 Map 中存储属性
把属性值存储在 map 中是一种常见的使用方式，这种操作经常出现在解析 JSON 或者其它动态的操作中。这种情况下你可以使用 map 来代理它的属性。

```
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
}

fun main(args: Array<String>) {
    val user = User(mapOf(
            "name" to "John Doe",
            "age"  to 25
    ))
    println(user.name) // Prints "John Doe"
    println(user.age)  // Prints 25

}
```
输出
```
John Doe
25
```
var 属性可以用 MutableMap 代替只读的 Map
```
class MutableUser(val map: MutableMap<String, Any?>) {
    var name: String by map
    var age: Int     by map
}
```
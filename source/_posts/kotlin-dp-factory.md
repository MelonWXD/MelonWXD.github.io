---
title: Kotlin 设计模式—工厂模式  
date: 2017-09-27 20:38:32  
tags: [设计模式,Kotlin]  
categories:  
---
学习Kotlin之设计模式
<!-- more -->

## 简单工厂模式
隐藏对象创建细节，用户只需要知道自己要什么，把参数传递给工厂，工厂负责对象的创建就给你什么。

```kotlin

/**
 * TV 抽象产品
 * HairTV HisenseTV 具体对象
 * SimpleTVFactory 简单工厂（制造TV的工厂）
 */


interface TV {
    fun play()
}

class HaierTV : TV {
    override fun play() {
        println("Haier TV is playing")
    }
}

class HisenseTV : TV {
    override fun play() {
        println("Hisense TV is playing")
    }
}

class SimpleTVFactory constructor(bandName: String) {
    var bandName: String = bandName

    fun produce(): TV {
        when (bandName) {
            "Haier" -> return HaierTV()
            "Hisense" -> return HisenseTV()
            else -> return object : TV {
                override fun play() {
                    println("Non-Band TV is playing")
                }
            }
        }
         
    }
}

fun main(args: Array<String>) {
    var haierFactory: SimpleTVFactory = SimpleTVFactory("Haier")
    var hisenseFactory: SimpleTVFactory = SimpleTVFactory("Hisense")
    var melonFactory: SimpleTVFactory = SimpleTVFactory("Dongua")
    haierFactory.produce()
    hisenseFactory.produce()
    melonFactory.produce()
}
```

## 工厂方法模式
随着业务的增加，用户可能不止需要海尔和海信的TV了，还需要美的TV，ok，没问题，在SimpleTVFactory的when语句里增加一个"Media"条件。然后小米、乐视各种TV都来了。不仅你改的累得慌，还违背了面向对象的开闭原则。
这时候就有了工厂方法模式，不仅产品抽象，工厂类也抽个接口出来，这样当你要新增类型的时候，只需要实现工厂类借口，实现具体的代码逻辑即可。
```kotlin
interface TV {
    fun play()
}

interface TVFactory {
    fun produce():TV
}

class HaierTV : TV {
    override fun play() {
        println("Haier TV is playing")
    }
}

class HaierTVFactory:TVFactory{
    override fun produce(): TV {
        println("HaierTVFactory is producing")
         return HaierTV()
    }
}

class HisenseTV : TV {
    override fun play() {
        println("Hisense TV is playing")
    }
}

class HisenseTVFactory:TVFactory{
    override fun produce(): TV {
        println("HisenseTVFactory is producing")
        return HisenseTV()
    }
}


fun main(args: Array<String>) {
    val tv:TV
    val tvF:TVFactory
    tvF = HisenseTVFactory()
    tv = tvF.produce()
    tv.play()
}
```

## 抽象工厂模式
随着竞争越来越激烈，各大厂商于是开始向CPU行业(这个CPU是搭载在TV上面的，就是说这两者之间是有关联的，你想，要是没有关联又何必放到同样的工厂下面提高耦合呢)进攻。
```kotlin
interface TV {
    fun play()
}
interface CPU{
    fun work()
}

interface MultiFactory {
    fun produceTV():TV
    fun produceCPU():CPU
}


class HaierTV : TV {
    override fun play() {
        println("HaierTV is playing")
    }
}
class HaierCPU : CPU {
    override fun work() {
        println("HaierCPU is working")
    }
}

class HaierFactory:MultiFactory{
    override fun produceTV(): TV {
        println("HaierFactory is producing TV")
        return HaierTV()
    }

    override fun produceCPU(): CPU {
        println("HaierFactory is producing CPU")
        return HaierCPU()
    }
}


class HisenseTV : TV {
    override fun play() {
        println("Hisense TV is playing")
    }
}


class HisenseCPU : CPU {
    override fun work() {
        println("HisenseCPU is working")
    }
}


class HisenseFactory:MultiFactory{

    override fun produceTV(): TV {
        println("HisenseFactory is producing TV")
        return HisenseTV()
    }

    override fun produceCPU(): CPU {
        println("HisenseFactory is producing CPU")
        return HisenseCPU()
    }
}



fun main(args: Array<String>) {
    val tv:TV
    val cpu:CPU
    val factory1:MultiFactory
    val factory2:MultiFactory
    factory1 = HaierFactory()
    factory2 = HisenseFactory()
    tv = factory1.produceTV()
    tv.play()
    cpu = factory2.produceCPU()
    cpu.work()



}

```
### 哎，感觉对抽象工厂不大懂，先留着








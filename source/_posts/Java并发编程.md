---
title: Java并发编程  
date: 2018-03-03 13:46:17
tags: [并发编程]  
categories: Java
---
总结Java并发编程中涉及到的一些知识

<!-- more -->  



# volatile

## 从双重检查锁加volatile引发的思考

一般我写单例模式是这样的

```java
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public static Singleton getInstance(){
        if(singleton == null){
            synchronized(Singleton.class){
                if(singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

今天发现别人是这样写的

```java
...
private volatile static Singleton singleton;
...
```

这就产生了一个问题，为什么用了`synchronized`之后，还要用`volatile`来修饰singleton呢？

还好知乎在这个问题上有[讨论](https://www.zhihu.com/question/56606703/answer/149721562)

- 防止指令的重排，使得出现先赋值，再初始化的情况，而其他线程在第一次判断`singleton == null`的时候，就直接返回了一个尚未初始化完成的singleton
- 操作的可见性，让抢到锁的线程1在创建完singleton实例之后，要把singleton从线程1的工作内存更新回主存


上面就介绍了volatile的有序性和可见性两种，volatile还有个原子性，**但volatile只能保证对单次读/写的原子性**，原子性是指一个操作是不可中断的，要么全部执行成功要么全部执行失败。

```java
x = 10;         //原子操作  将数值10赋值给x
y = x;         //非原子操作  先读取x的值 然后把值赋给y
x++;           //非原子操作 先读取x的值 加1 把值赋给x
x = x + 1;     //同上
```

```java
public class VolatileTest {
    volatile int i;
    public void addI(){
        i++;
    }

    public static void main(String[] args) throws InterruptedException {
        final  VolatileTest test = new VolatileTest();
        for (int n = 0; n <2000; n++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.addI();
                }
            }).start();
        }

        Thread.sleep(20000);

        System.out.println(test.i);
    }
}
```

所以上面的代码输出的值不一定为2000，因为`i++`操作不具有原子性



# synchronized

这个就比volatile牛逼多了，synchronized可以修饰方法、代码块，并且也保证了原子性与可见性。就上面i++的demo中把方法`addI`用synchronized修饰一下保证每次的值都是2000。

两者主要差别

volatile是告知jvm，这个变量在寄存器中的值是不确定的，当某个线程修改了变量的值后会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时， 发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读。

synchronized是只允许持有锁的线程对这个变量进行操作，当前线程讲值写回主存之后，其他线程才会去读取(这个应该是这样的。。)，就不会出现先读取值，然后再被其他线程修改，出现不一致的情况了。



# Lock



# yield、join

[看这里](http://www.cnblogs.com/paddix/p/5381958.html)
---
title: RxJava (四)
date: 2017-11-01 19:21:30
tags: [RxJava] 
categories: Android  
---
Backpressure背压和Flowable
<!-- more -->
## Cold and Hold Observables
Rxjava的Observable有Hot和Cold两种，一个是火辣美眉，主动出击，不管有没有Observer订阅，Observer中途订阅也无法接受到之前的数据。一个是高冷女神，坐等撩拨，只有Observer订阅的时候，才会开始发送数据，而且如果有多个Observer中途订阅，Cold Observable也会把之前的数据流依次发送给Observer。


### Cold Observables
RxJava创建型操作符[Interval](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Interval.html)就是个典型的Cold Observable，他会根据你给的时间间隔，来依次发送0,1,2...直接看代码吧
```java
    public static void main(String[] args) throws Exception {

        Observable<Long> cold = Observable.interval(200, TimeUnit.MILLISECONDS);
        try {

            cold.subscribe(i -> System.out.println("First: " + i));
            Thread.sleep(400);
            cold.subscribe(i -> System.out.println("Second: " + i));
            Thread.sleep(1000);
        } catch (Exception e) {
        }

    }
```
输出
```
First: 0
First: 1
First: 2
Second: 0
First: 3
Second: 1
First: 4
Second: 2
```
该Observable 200ms发送1次数据，在400ms之后，新增一个Observer，观察日志发现Second也是从0开始接收的。

### Hot Observables
使用 publish 操作函数可以把 Cold Observable 转化为 ConnectableObservable，实际上就是一个Hot Observable，代码如下
```java
   public static void main(String[] args) throws Exception {

        ConnectableObservable hot = Observable.create(observableEmitter -> {
            new Thread(()->{
                int i = 0;
                while (i < 20) {
                    observableEmitter.onNext(i++);
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                    }
                }
            }).start();

        }).publish();

        hot.connect();
        hot.subscribe(i -> System.out.println("First: " + i));
        Thread.sleep(400);
        hot.subscribe(i -> System.out.println("Second: " + i));
        Thread.sleep(10000);
    }
```
输出
```
First: 1
First: 2
First: 3
Second: 3
First: 4
Second: 4
First: 5
Second: 5
First: 6
Second: 6
First: 7
Second: 7
```
看见没，这回Second中途插进来，只能接收到当时发送的3,1和2都没啦。  
对Hot Observable来说，同一时刻Observer收到的都是相同的数据。

## 背压BackPressure 
BackPressure是什么？  
官方的原文是`in order to alleviate the problems caused when a quickly-producing Observable meets a slow-consuming observer.`简而言之，就是一种缓解**Observable发送速度与Observer处理速度不匹配**的一种策略。

### MissingBackpressureException
在**RxJava1**中，如果Observable发送事件的速度远远超过了Observer的接收处理速度，即上下游速率不匹配的时候，就会跑出一个 MissingBackpressureException，错误代码如下：
```java
        Observable
                .interval(20, TimeUnit.MILLISECONDS)
                .take(Integer.MAX_VALUE)
                .subscribe(cLong->{
                    Thread.sleep(1000);
                    System.out.println(cLong);
                });
```
上游的Observable每隔20ms就发送1个值，而下游1000ms才处理1次。没被处理的数据就会被存到内存中，根据Rxjava1的源码得知，当内存中暂存的数据超过128个的时候，就会抛出MissingBackpressureException了。
### Flowable
为什么上面要强调在RxJava1中呢，因为在RxJava2中Observable中没背压这个概念，官方引入了FLowable这个类来专门做背压的处理。上述同样的代码，只会引发OOM而不是MissingBackpressureException。  
来看看FLowable的简单使用：
```java
    public static void main(String[] args) throws Exception {
        System.out.println(Flowable.bufferSize());
        
        Flowable.create(flowableEmitter -> {
                    for (int i = 0; i < 1000; i++) {
                        flowableEmitter.onNext(i);
                    }
                }
                , BackpressureStrategy.DROP)
                .observeOn(Schedulers.newThread())
                .subscribeOn(Schedulers.io())
                .subscribe(cLong -> {
                    Thread.sleep(100);
                    System.out.println("recv" + cLong);
                });


        Thread.sleep(100000);
    }
```
输出
```
128
recv0
recv1
recv2
recv3
recv4
recv5
...
recv124
recv125
recv126
recv127
```
内部flowableEmitter的数据流来自于一个Observable，也是起到interval的效果。第一行输出FLowable内置的BufferSize缓存区大小为128，同时设置背压策略为`BackpressureStrategy.DROP`，所以在发送和接收速率不匹配的时候，上游只会缓存128个值，剩下的DROP掉了，这也说明了为什么输出只到recv127。



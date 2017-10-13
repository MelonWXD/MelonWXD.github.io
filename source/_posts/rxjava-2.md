---
title: RxJava (二)  
date: 2017-10-13 16:44:54
tags: [RxJava] 
categories: Android  
---
RxJava 线程调度
<!-- more -->
## RxJava线程调度
线程调度就是允许观察者和被观察者处在不同的线程中，通过RxJava内置的调度器可以很轻松的完成这些功能。
开发会用到的主要线程选项：
- Schedulers.io() 代表io操作的线程, 通常用于网络,读写文件等io密集型的操作
- Schedulers.computation() 代表CPU计算密集型的操作, 例如需要大量计算的操作
- Schedulers.newThread() 代表一个常规的新线程
- AndroidSchedulers.mainThread() 代表Android的主线程  

### 基本使用
subscribeOn() 指定的是被观察者所在的线程
observeOn() 指定的是观察者所在的线程
如果不说明，那么默认是在创建实例的线程
```
observable.subscribeOn(Schedulers.newThread())                    
            .observeOn(AndroidSchedulers.mainThread())             
            .subscribe(observer);

```

### 多次指定观察者线程
多次subscribeOn指定的线程只有第一次指定的有效,  其余的会被忽略。
多次observerOn，则是最后一次指定的有效，每调用一次observeOn()，对应的观察者就会改变到该线程，例子如下：
```java
 Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                Log.d(TAG, "Observable thread is :" + Thread.currentThread().getName());
                emitter.onNext(1);
                emitter.onNext(2);
            }
        });

        Consumer<Integer> consumer = new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                Log.d(TAG, "consumer1 thread is :" + Thread.currentThread().getName());
                Log.d(TAG, "consumer1 onNext: " + integer);
            }
        };

        observable.subscribeOn(AndroidSchedulers.mainThread())
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .observeOn(Schedulers.io())
                .doOnNext(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.d(TAG, "consumer1 thread is :" + Thread.currentThread().getName());
                        Log.d(TAG, "consumer1 onNext: " + integer);
                    }
                })
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.d(TAG, "consumer2 thread is :" + Thread.currentThread().getName());
                        Log.d(TAG, "consumer2 onNext: " + integer);
                    }
                });
```
输出：
```
10-13 20:48:23.222 18592-18592/com.dongua.rxand D/MainActivity: Observable thread is :main
10-13 20:48:23.266 18592-18718/com.dongua.rxand D/MainActivity: consumer1 thread is :RxCachedThreadScheduler-2
10-13 20:48:23.267 18592-18718/com.dongua.rxand D/MainActivity: consumer1 onNext: 1
10-13 20:48:23.268 18592-18718/com.dongua.rxand D/MainActivity: consumer1 thread is :RxCachedThreadScheduler-2
10-13 20:48:23.268 18592-18718/com.dongua.rxand D/MainActivity: consumer1 onNext: 2
10-13 20:48:23.270 18592-18719/com.dongua.rxand D/MainActivity: consumer2 thread is :RxNewThreadScheduler-1
10-13 20:48:23.270 18592-18719/com.dongua.rxand D/MainActivity: consumer2 onNext: 1
10-13 20:48:23.271 18592-18719/com.dongua.rxand D/MainActivity: consumer2 thread is :RxNewThreadScheduler-1
10-13 20:48:23.271 18592-18719/com.dongua.rxand D/MainActivity: consumer2 onNext: 2
```
分析：  
- 两次调用subscribeOn，先是mainThread，再是ioThread，打印出来是main。  
- 在consumer1订阅之前，两次调用observerOn，先是mainThread，再是ioThread，打印出来是RxCachedThreadScheduler-2。   
- 在consumer2订阅之前，调用observerOn，使用newThread来通知，consume2打印出来是与consumer1不同的RxNewThreadScheduler-1。  

---
title: RxJava (一)  
date: 2017-10-09 21:59:07  
tags: [RxJava,Retrofit] 
categories: Android  
---
RxJava系列
<!-- more -->

## 观察者模式和RxJava的异同
先来快速的写一个观察者模式的实现，大致如下：
```java
public interface Observer {
    void update(Event event);
}
public interface Observable{
    void subscribe(Observer observer);
    void unsubscribe(Observer observer);
    void notifyObserver(Event event);
}
```
这里有观察者Observer和被观察者Observable，Observable有订阅和取消订阅，以及通知Observer的notify方法，以及参数Event。
这里就囊括了RxJava的四个基本要素：
- 观察者Observer
- 被观察者Observable
- 订阅subscribe
- 事件Event  

而RxJava在此基础上，还定义了2个特殊事件：
- onCompleted
- onError  

## RxJava基本实现
### RxJava1.0
#### 观察者Observer/Subscriber
```java
//Observer<String> observer = new Observer<String>() 
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");立连接
    }
};
```
Subscriber相对于Observer新增了2个方法：
- onStart
- unsubscribe　

#### 被观察者Observable
```java
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Event1");
        subscriber.onNext("Event2");
        subscriber.onNext("Event3");
        subscriber.onCompleted();
    }
});
```
#### 订阅subscrible
```java
observable.subscrible(subsricber)
```

### RxJava2.0
#### 观察者Observer/Subscriber
```java
Observer<Integer> observer = new Observer<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {
                Log.d(TAG, "subscribe");
            }

            @Override
            public void onNext(String value) {
                Log.d(TAG, value);
            }

            @Override
            public void onError(Throwable e) {
                Log.d(TAG, "error");
            }

            @Override
            public void onComplete() {
                Log.d(TAG, "complete");
            }
        };

```

#### 被观察者Observable
```java
Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext("Event1");
                emitter.onNext("Event2");
                emitter.onNext("Event3");
                emitter.onComplete();
            }
        });
```
#### 订阅subscrible
```java
observable.subscrible(subsricber)
```
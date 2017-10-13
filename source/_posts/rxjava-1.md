---
title: RxJava (一)  
date: 2017-10-09 21:59:07  
tags: [RxJava,Retrofit] 
categories: Android  
---
RxJava 基本用法
<!-- more -->
## 参考资料
[Rxjava 2.0](http://www.jianshu.com/u/c50b715ccaeb)  
[Rxjava 1.0](http://gank.io/post/560e15be2dca930e00da1083)
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
这里展示一下1.0的基本用法 后续都是基于rxjava2.0来演示
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
Observer<Integer> observer = new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                System.out.println("subscribe");
            }

            @Override
            public void onNext(String value) {
                System.out.println(value);
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("error");
            }

            @Override
            public void onComplete() {
                System.out.println("complete");
            }
        };

```

#### 被观察者Observable
```java
Observable<Integer> observable =Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
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
#### rxjava链式写法
```java
Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("Event1");
                emitter.onNext("Event2");
                emitter.onNext("Event3");
                emitter.onComplete();
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                System.out.println("subscribe");
            }

            @Override
            public void onNext(String value) {
                System.out.println(value);
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("error");
            }

            @Override
            public void onComplete() {
                System.out.println("complete");
            }
        });
```
输出如下：
```
subscribe
Event1
Event2
Event3
complete
```
### 1.0 2.0差别
#### ObservableEmitter
这里2.0在实例化Observable的时候，使用的是ObservableOnSubscribe这个类，并且重写onSubscribe方法，参数是ObservableEmitter（Emitter译为发射器）。通过调用emitter的onNext(T value)、onComplete()和onError(Throwable error)就可以分别发出next事件、complete事件和error事件，对应Observer的三个方法。
#### Disposable
在Observer的onSubscribe方法中，参数类型为Disposable，拿到这个参数，Observer可以随时调用dipose方法来结束本次订阅，尽管Observable的事件依然会按顺序发送直到接送，但Observer在调用dipose之后就停止了接收。
#### Consumer
在2.0中subscribe有多个重载方法
```
public final Disposable subscribe() {}
    public final Disposable subscribe(Consumer<? super T> onNext) {}
    public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError) {} 
    public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete) {}
    public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete, Consumer<? super Disposable> onSubscribe) {}
    public final void subscribe(Observer<? super T> observer) {}
```
拿来替换1.0中的Action0、Action1等，在1.0中通过参数个数来分别对应next、competed、error事件，代码如下：
```java
Action1<String> onNextAction = new Action1<String>() {
    // onNext()
    @Override
    public void call(String s) {
        Log.d(tag, s);
    }
};
Action1<Throwable> onErrorAction = new Action1<Throwable>() {
    // onError()
    @Override
    public void call(Throwable throwable) {
        // Error handling
    }
};
Action0 onCompletedAction = new Action0() {
    // onCompleted()
    @Override
    public void call() {
        Log.d(tag, "completed");
    }
};

// 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
observable.subscribe(onNextAction);
// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
observable.subscribe(onNextAction, onErrorAction);
// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
observable.subscribe(onNextAction, onErrorAction, onCompletedAction);
```
而在2.0中，使用方法也差不多，也是通过参数判断，只不过整合到一个Consumer类中
```java
        Consumer<String> nextConsumer = new Consumer<String>() {
            @Override
            public void accept(String string) throws Exception {
                System.out.println( "onNext: " + string);
            }
        };
        Consumer<Disposable> disposableConsumer = new Consumer<Disposable>() {
            @Override
            public void accept(Disposable disposable) throws Exception {
                System.out.println( "Disposable");
            }

        };

        Action competedAction = new Action() {
            @Override
            public void run() throws Exception {
                System.out.println( "onComplete" );
            }
        };

        Consumer<Throwable> errorConsumer = new Consumer<Throwable>() {
            @Override
            public void accept(Throwable e) throws Exception {
                System.out.println( "onError: " + e.getMessage());
            }
        };


        Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("Event1");
                e.onComplete();
            }
        }).subscribe(nextConsumer,errorConsumer,competedAction,disposableConsumer);
```
输出：
```
Disposable
onNext: Event1
onComplete
```


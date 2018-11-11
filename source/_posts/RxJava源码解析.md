---
title: RxJava
date: 2018-03-28 10:17:06
tags: [RxJava]  
categories: Android
---

RxJava流程分析

<!-- more -->  

# 基本使用

```java
        Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("haha");
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String s) {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onComplete() {

            }
        });
```

## create

无脑跟进法

```java
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```

通过`Observable.create()`返回一个Observable子类ObservableCreate的实例。

## subscribe

跟进`subscribe`，ObservableCreate没有重写该方法，在父类Observable中查找：

```java
    public final void subscribe(Observer<? super T> observer) {
        try {
          
            observer = RxJavaPlugins.onSubscribe(this, observer);
            subscribeActual(observer);
          
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            throw npe;
        }
    }
```
可以再来看看`RxJavaPlugins.onSubscribe`搞什么鬼

```java
    public static <T> Observer<? super T> onSubscribe(@NonNull Observable<T> source, @NonNull Observer<? super T> observer) {
        BiFunction<? super Observable, ? super Observer, ? extends Observer> f = onObservableSubscribe;
        if (f != null) {
            return apply(f, source, observer);
        }
        return observer;
    }
```

可以发现包括上面的`onAssembly`，RxJavaPlugins做的都是一些额外包装的工作，类似于hook的功能，如果分析朱脉络的可以先不管这个东西。

在看看`subscribeActual(observer);`，在Observable中是抽象方法，说明针对不同的创建操作符，如create、just等，都继承于Observable，并重写了subscribeActual来实现具体的逻辑。

```java
    protected abstract void subscribeActual(Observer<? super T> observer);
```

跟进`ObservableCreate.subscribeActual`

```java
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```

先讲observer包装为CreateEmitter

```java
static final class CreateEmitter<T>  extends AtomicReference<Disposable>
    					implements ObservableEmitter<T>, Disposable {
		...
        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }
        ...
}
```

再通过`observer.onSubscribe(parent);`回调我们传入的observer的`void onSubscribe(@NonNull Disposable d)`方法，通常我们就是在这里拿到disposable对象来中断流的发送。

然后通过`source.subscribe(parent);`回调我们传入的Observable的`void subscribe(@NonNull ObservableEmitter<T> e) throws Exception`方法，通常我们是在这里拿到ObservableEmitter对象，即上述的parent对象，来调用他的onNext等方法。

再回到基本使用

```java
Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("haha");
            }
        })
  			...
}
```

在create传入的ObservableOnSubscribe就是ObservableCreate中的source，subscribe方法中的参数`ObservableEmitter<String> e`就是ObservableCreate的方法subscribeActual中`source.subscribe(parent);`的参数`parent`，即`new CreateEmitter<T>(observer);`。

e = parent = new CreateEmitter<T>(observer)

所以来看看parent即CreateEmitter的onNext方法

```java
        @Override
        public void onNext(T t) {
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }
```

就是这么回调到`observer.onNext(t);`中滴，流程很清楚。

# 线程调度

```java
  Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("haha");
            }
        }).subscribeOn(Schedulers.io())
    	.observeOn(AndroidSchedulers.mainThread())
   		.subscribe(new Observer<String>() {
      		...
    	});

```

## subscribeOn 

“订阅”所在的线程，即Observable中subscribe方法所在线程，即上游所在线程。

无脑跟进法，启动。。

```java
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }

```

跟上面分析的create其实差不多，在没有Function的情况下，onAssembly的返回值就是他的参数。再回顾一下create方法，返回的是包装了`source`的ObservableCreate。

```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```

那么在`subscribeOn`方法中，onAssembly参数中构造的ObservableSubscribeOn的参数this就是这个ObservableCreate，同时还有一个scheduler。

```java
    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }
```

先不考虑observerOn，直接按照上面分析的订阅流程来，会调用到ObservableSubscribeOn的`subscribeActual`方法

```java
    @Override
    public void subscribeActual(final Observer<? super T> s) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

        s.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
```

比ObservableCreate的subscribeActual多了一个`parent.setDisposable…`，跟进scheduler.scheduleDirect之前先看一下scheduler具体是哪个子类

```java

    public static Scheduler newThread() {
        return RxJavaPlugins.onNewThreadScheduler(NEW_THREAD);
    }

    public static Scheduler onNewThreadScheduler(@NonNull Scheduler defaultScheduler) {
        Function<? super Scheduler, ? extends Scheduler> f = onNewThreadHandler;
        if (f == null) {
            return defaultScheduler;
        }
        return apply(f, defaultScheduler);
    }

```

返回的是传入的NEW_THREAD,那么在看看这个NEW_THREAD又是个啥

```java
NEW_THREAD = RxJavaPlugins.initNewThreadScheduler(new NewThreadTask());

public static Scheduler initNewThreadScheduler(@NonNull Callable<Scheduler> defaultScheduler) {
    
        Function<...> f = onInitNewThreadHandler;
        if (f == null) {
            return callRequireNonNull(defaultScheduler);
        }
        
    }
static Scheduler callRequireNonNull(@NonNull Callable<Scheduler> s) {
        try {
            
            return ObjectHelper.requireNonNull(s.call(), "...");
            
        } catch (Throwable ex) {
            
        }
    }
public static <T> T requireNonNull(T object, String message) {
        if (object == null) {
            throw new NullPointerException(message);
        }
        return object;
    }
```

看上面代码就知道返回的是`new NewThreadTask().call()`

```java
  static final class NewThreadTask implements Callable<Scheduler> {
        @Override
        public Scheduler call() throws Exception {
            return NewThreadHolder.DEFAULT;
        }
    }
    static final class NewThreadHolder {
        static final Scheduler DEFAULT = new NewThreadScheduler();
    }
    public NewThreadScheduler() {
        this(THREAD_FACTORY);
    }
```

oj8k,上面线程调度传入的Scheduler.newThread()即为NewThreadScheduler实例。

再回到上面ObservableSubscribeOn的subscribeActual方法中跟进`scheduler.scheduleDirect(new SubscribeTask(parent)`

```java
public Disposable scheduleDirect(@NonNull Runnable run) {
        return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}

public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        final Worker w = createWorker();

        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        DisposeTask task = new DisposeTask(decoratedRun, w);

        w.schedule(task, delay, unit);

        return task;
    }
```

createWorker由NewThreadScheduler重写

。。就写到这吧，不想分析了，后面就是封装runnable submit到线程池执行，溜了溜了，准备上班

runnable就是

```java

    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
```

其实就是把source.subscribe(parent);放到县城里执行达到调度的效果嘛。
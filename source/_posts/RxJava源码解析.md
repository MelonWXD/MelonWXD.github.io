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
    		...
      }).subscribeOn(Schedulers.io())
    	.observeOn(AndroidSchedulers.mainThread())
   		.subscribe(new Observer<String>() {
      		...
    	});

```

//todo

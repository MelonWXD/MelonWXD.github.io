---
title: LeakCanary 源码分析  
date: 2018-03-07 13:38:12
tags: [性能优化]  
categories: Android框架
---
LeakCanary检测内存泄露原理
<!-- more -->  

# WeakReference和ReferenceQueue

在jdk1.2之后，Java对引用的概念进行了扩充，将引用分为强引用(Strong Reference)、软引用(Soft Reference)、弱引用(Weak Reference)、虚引用(Phantom Reference)四种。

平常加载图片的时候，经常需要把经常显示的图片缓存在内存中来提高加载效率和程序的性能。想象一下如果在jdk1.1时代，我们的Bitmap缓存对象肯定通过强引用来缓存，就无法被GC了，如果缓存一多整个程序就gg了。所以WeakReference就出现了，当一个对象仅仅被WeakReference指向， 而没有任何其他StrongReference指向的时候， 如果GC运行， 那么这个对象就会被回收。所以厂常见的缓存算法LRUCache里面都是通过WeakReference来持有对象的。

我们希望当一个对象被gc掉的时候通知用户线程，进行额外的处理时，就需要使用引用队列了。ReferenceQueue即这样的一个对象，当一个obj被gc掉之后，其相应的包装类，即WeakReference对象会被放入queue中。我们可以从queue中获取到相应的对象信息，同时进行额外的处理。

具体的用法如下：

```java
public static void main(String[] args) {
        ReferenceQueue<byte[]> referenceQueue = new ReferenceQueue();
        Object value = new Object();
        Map<Object, Object> map = new HashMap<>();
        for(int i = 0;i < 10000;i++) {
            byte[] bytes = new byte[1024*1024];
            WeakReference<byte[]> weakReference = new WeakReference<byte[]>(bytes, referenceQueue);
            map.put(weakReference, value);
        }
        System.out.println("map.size->" + map.size());

        Thread thread = new Thread(() -> {
            try {
                int cnt = 0;
                WeakReference<byte[]> k;
                while((k = (WeakReference) referenceQueue.remove()) != null) {
                    System.out.println((cnt++) + "回收了:" + k);
                }
            } catch(InterruptedException e) {
                //结束循环
            }
        });
        thread.setDaemon(true);
        thread.start();
    }
```

# LeakCanary

## 使用

最简单的使用就是在Application的onCreate中一行即搞定

```java
    @Override
    public void onCreate() {
        super.onCreate();
        LeakCanary.install(this);
    }
```

## 原理

采用无脑跟进法分析！

```java
public static RefWatcher install(Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }

public static AndroidRefWatcherBuilder refWatcher(Context context) {
    return new AndroidRefWatcherBuilder(context);
  } 
```

通过AndroidRefWatcherBuilder的`buildAndInstall`来创建一个RefWatcher实例

```java
  public RefWatcher buildAndInstall() {
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      LeakCanary.enableDisplayLeakActivity(context);
      ActivityRefWatcher.install((Application) context, refWatcher);
    }
    return refWatcher;
  }
```

通过`ActivityRefWatcher.install`来创建ActivityRefWatcher实例并监听Activity的生命周期

```java
public static void install(Application application, RefWatcher refWatcher) {
    new ActivityRefWatcher(application, refWatcher).watchActivities();
  }
```

```java
  public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
  }


  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new Application.ActivityLifecycleCallbacks() {
		...
        @Override public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
      };

```

通过lifecycleCallbacks来在Activity的生命周期中回调，实际上只在`onActivityDestroyed`中做了处理，调用了 ActivityRefWatcher的onActivityDestroyed(activity)方法。

```java
  void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);
  }
```

跟进

```java
public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }

  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();//生成唯一的key，来定位watchedReference
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
  }

	private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

在watch中把被观察对象`watchedReference`封装成带有key值和带引用队列的KeyedWeakReference对象，然后在ensureGoneAsync中的ensureGone进行分析泄露。

### 泄露分析

```java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks. 
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc();
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }
```

先来看看`removeWeaklyReachableReferences`

```java
  private void removeWeaklyReachableReferences() {
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```

KeyedWeakReference是WeakReference子类，这里用到的就是我上面介绍的 通过WeakReference加上ReferenceQueue来实现对已经被gc的对象做额外处理。如果`queue.poll()!=null`成立，说明被观察的对象已经被gc掉了，因为只有被gc了相应的包装类WeakReference的实例才会被添加到ReferenceQueue中。

然后通过`gone`来判断是否有泄露

```java
 private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```

如果有泄露，就手动触发一下gc，再来判断有没有泄露。如果确认泄露了，那么就开始进入内存分析。

### 内存快照分析

```java
if (!gone(reference)) {
  long startDumpHeap = System.nanoTime();
  long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

  File heapDumpFile = heapDumper.dumpHeap();
  if (heapDumpFile == RETRY_LATER) {
    // Could not dump the heap.
    return RETRY;
  }
  long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
  heapdumpListener.analyze(
      new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
          gcDurationMs, heapDumpDurationMs));
}
```
确定泄露之后就要分析内存快照，根据上面唯一的key来定位，项目内部又使用了另一框架[haha](https://github.com/square/haha)来分析。

分析的代码如下：

```java
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
```

构造器传入的Listener如下

```java
  public static RefWatcher install(Application application) {
    return refWatcher(application)
        .listenerServiceClass(DisplayLeakService.class)//设置Listener
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  } 

	public AndroidRefWatcherBuilder listenerServiceClass(Class<...> listenerServiceClass) {
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }
```

所以上面的heapdumpListener就是ServiceHeapDumpListener的实例，看看这类的analyze方法

```java
  @Override public void analyze(HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
```

实际上就是启动一个IntentService来进行内存分析操作

```java
public final class HeapAnalyzerService extends IntentService {

  private static final String LISTENER_CLASS_EXTRA = "listener_class_extra";
  private static final String HEAPDUMP_EXTRA = "heapdump_extra";

  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    context.startService(intent);
  }

  public HeapAnalyzerService() {
    super(HeapAnalyzerService.class.getSimpleName());
  }

  @Override protected void onHandleIntent(Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
}
```

跟进

```java
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
      Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
      return failure(exception, since(analysisStartNanoTime));
    }

    try {
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      HprofParser parser = new HprofParser(buffer);
      Snapshot snapshot = parser.parse();
      deduplicateGcRoots(snapshot);

      Instance leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        return noLeak(since(analysisStartNanoTime));
      }

      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }
```

上面的checkForLeak方法就是输入.hprof，输出分析结果，主要有以下几个步骤：

1.把.hprof转为Snapshot，这个Snapshot对象就包含了对象引用的所有路径

2.精简gcroots,把重复的路径删除，重新封装成不重复的路径的容器

3.找出泄漏的对象

4.找出泄漏对象的最短路径



# 参考

[ReferenceQueue的使用](http://www.cnblogs.com/dreamroute/p/5029899.html)


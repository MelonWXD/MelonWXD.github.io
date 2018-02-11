---
title: Android多线程学习  
date: 2018-02-8 14:30:10
tags: [多线程]  
categories: Android
---
Android中的多线程学习
<!-- more -->  

# 综述

Android提供了四种常用的操作多线程的方式，分别是：
1. HandlerThread
2. AsyncTask
3. IntentService
4. ThreadPoolExecutor

> - **HandlerThread:** 为某些回调方法或者等待某些任务的执行设置一个专属的线程，并提供线程任务的调度机制。
> - **AsyncTask:** 为 UI 线程与工作线程之间进行快速的切换提供一种简单便捷的机制。适用于当下立即需要启动，但是异步执行的生命周期短暂的使用场景。
> - **IntentService:** 适合于执行由 UI 触发的后台 Service 任务，并可以把后台任务执行的情况通过一定的机制反馈给 UI。
> - **ThreadPool:** 把任务分解成不同的单元，分发到各个不同的线程上，进行同时并发处理。


HandlerThread在之前的一篇文章里已经讲过了，ThreadPoolExecutor是Java中处理多线程的方法，下面从源码角度分析AsyncTask和IntentService的原理

# AsyncTask

## 基本用法

```java
	//继承抽象类 实现抽象方法doInBackground 
    class MyAsyncTask extends AsyncTask<Void, Integer, String> {

        @Override
        protected String doInBackground(Void... voids) {
            publishProgress(10);
          	//do some work
            return null;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
        }

    }

	//在UI线程中创建实例 执行
	MyAsyncTask asyncTask = new MyAsyncTask();
    asyncTask.execute();
```

AsyncTask是个抽象类，我们需要实现`doInBackground`，通常我们还会实现`onProgressUpdate`来做一个进度的回调与`onPostExecute`实现结果的回调。在`doInBackground`内部通过`publishProgress`来通知`onProgressUpdate`，怎么通知会放到下面doInBackground的分析中讲解。

```java
public abstract class AsyncTask<Params, Progress, Result> {...}
```

还需要传入三个泛型来分别表示执行参数Params，进度参数Progress，结果参数Result。

Params对应着`execute`和`doInBackground`的参数类型

```java
 public final AsyncTask<Params, Progress, Result> execute(Params... params){...}
 protected abstract Result doInBackground(Params... params);
```

Progress对应着`onProgressUpdate`和`publishProgress`的参数类型

```java
protected void onProgressUpdate(Progress... values) {...}
protected final void publishProgress(Progress... values) {...}
```

Result对应着`onPostExecute`的参数类型

```java
protected void onPostExecute(Result result) {...}
```



知道了每个方法对应着什么样的作用，用起来就很简单了。



## 源码分析

### 构造函数

不管怎么调，最后都是来到这个方法

```java
public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
				...
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                ...
            }
        };
    }

    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }

```

当我们调用的是无参的构造函数时，mHandler就是InternalHandler的实例，下面在doInBackground中会详细说明。

mWorker是一个基于Callable接口封装的WorkerRunnable的实例，也就多了个储存执行参数的数组而已

```java
private final WorkerRunnable<Params, Result> mWorker;
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
```

mFuture就是`FutureTask<Result>`的实例

### execute

```java
    @MainThread
	public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

有2个值得注意的点：

- @MainThread 意味着excute需要在主线程执行
- 检测mStatus来抛出异常，说明一个AsyncTask**对象**只能执行一次

检查和赋值完状态，调用了`onPreExecute`，用户可以重写这个方法来做一些预处理。

然后把执行的参数赋值给mWorker，然后调用`exec.execute(mFuture);`，结合调用关系，知道exec就是sDefaultExecutor

### Executor

sDefaultExecutor是SerialExecutor的实例，SerialExecutor实现了Executor，内部维护一个双端队列mTasks与一个代表当前任务的mActive。

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

代码很简单，执行上述的`exec.execute(mFuture);`实际上就是把mFuture经过一个匿名的Runnable包装后添加到sDefaultExecutor的mTasks中，然后从mTasks中取出队列头的Runnable任务并通过线程池THREAD_POOL_EXECUTOR来执行。

由此可以知道AsyncTask是串行执行任务的。执行的任务是被包装后的mFuture，实际上就是mFuture的callable变量，即mWork。

如果你不想串行执行任务，直接使用内部的线程池THREAD_POOL_EXECUTOR来执行，那么可以直接调用`executeOnExecutor`方法，指定Executor来执行任务

```java
asyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
```



### doInBackground

在回到构造函数中看看mHandler和mWork的创建过程。

```java
    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

mHandler是InternalHandler的实例，针对`MESSAGE_POST_RESULT`和`MESSAGE_POST_PROGRESS`做了处理，分别对应这**结果**的通知和**进度**的通知。

再来看看mWork。

```java
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
```

mWorker最后会被Executor调用，call方法中调用了由用户重写的doInBackground，执行具体的工作任务。

那么为什么在doInBackground中执行`publishProgeress`就可以回调通知到`onProgressUpdate`呢，看了上面InternalHandler的封装心里基本也有点数了

```java
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

  private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

`publishProgress`发送了`MESSAGE_POST_PROGRESS`

同样的在mWorker的call方法中finally块执行了`postResult`，发送了`MESSAGE_POST_RESULT`



# IntentService

## HandlerThread 

还是再讲讲HandlerThread吧，发现又有点忘了。

```java
public class HandlerThread extends Thread {
        @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
  
      public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}

```

源码没什么好分析的，一个线程，封装好Looper，就一直loop循环等待消息到来直到quit。

跟一般的线程比起来就是提高了与主线程的交互能力，主线程可以在HandlerThread启动之后，通过`getLooper`新建一个Handler来与HandlerThread建立起连接，通过`sendMessage`来通知HandlerThread线程，并重写这个Handler的`handleMessage`来做对应的处理。

实例：

```java
    Handler mUIHandler = new Handler();
    Handler mBackHandler;
    HandlerThread mBachHandlerThread;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        mBachHandlerThread = new HandlerThread("test");
        mBachHandlerThread.start();
        mBackHandler = new Handler(mBachHandlerThread.getLooper()){
            @Override
            public void handleMessage(Message msg)  {
                Thread.sleep(1000); //做一些耗时操作
              
                mUIHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        //更新ui
                    }
                });
            }
        };
        //通知mBachHandlerThread
        mBackHandler.sendEmptyMessage(1);
    }

    @Override
    protected void onPause() {
        super.onPause();
        mBackHandler.removeCallbacksAndMessages(null);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mBachHandlerThread.quit();
    }
```



## 基本用法

```java
public class MyIntentService extends IntentService {
    public MyIntentService(String name) {
        super(name);
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        try {
            int progeress = 0;
            while (progeress < 100) {
                progeress++;

                Thread.sleep(500);
                sendBroadcast(progeress);//通过LocalBroadcastManager发送广播给UI线程来更新UI
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void sendBroadcast(int progeress) { ... }
}

```

继承IntentService，实现构造方法和`onHandleIntent`即可，剩下的按照普通Service处理（清单注册，intent启动）即可。

下面来分析一下内部实现。

## onCreate

```java
    @Override
    public void onCreate() {
 
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```

跟上述我们使用HandlerThread一样的用法，让mServiceHandler与HandlerThread建立起关联。

## onStart

```java
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```

已启动就使用mServiceHandler发了个消息

```java
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

回调了`onHandleIntent`，该方法由用户实现，并且调用了`stopSelf`，体现IntentService用完就走的设计思路。

# ThreadPoolExecutor

再补充一下线程池中相关。ThreadPoolExecutor提供了4个构造方法，最后都会调用到下面这个。

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

挑几个比较重要的拿出来讲一下。

BlockingQueue<Runnable> workQueue，主要有三个类实现了BlockingQueue这个接口：

- SynchronousQueue
- LinkedBlockingDeque
- ArrayBlockingQueue

ThreadFactory threadFactory，提供对线程一些属性定制的操作

## 执行

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         * 
         1. 检查能不能添加
         
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         2.添加后的二次检查
         
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         3.不能添加就reject
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```


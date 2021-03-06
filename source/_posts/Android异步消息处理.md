---
title: Android异步消息处理
date: 2017-12-26 20:47:13
tags: 多线程
categories: Android
---
源码角度分析Android异步消息处理机制
<!-- more -->
# 写在前面
最近公司变动频繁，心下略慌，刚才随便逛看到2个面试题：
1. 一个线程有几个Handler，如果一个线程有多个Handler，那么怎么系统怎么确定某个Message所回调到的handleMessage方法

2. 两个副线程可以使用Looper.prepare公用一个MessageQueue吗？如果可以怎么实现

下意识给出如下答案：

1. 一个线程可以有多个Handler。Handler在发送Message的时候会通过msg.target=this来把Handler和该Handler发送的消息关联起来。Message在被Looper提取处理的时候，会调用Message对应的tagert的dispatchMessage来，从而实现多个Handler回调的准确性。


2. 第一反应是不能的，首先Looper.prepare的方法是通过ThreadLocal来保证多线程的互不影响，也保证了Looper在每个线程中都是唯一的，他这个题目我首先就有点看不懂了。最后看他说，如果可以，怎么实现？兄弟，貌似不可以啊，如果不可以为什么还要问怎么实现？一点思路都没有，又一想到Handler这个源码每次都是看了忘，遂再次走读源码并做个笔记。

# Looper
```java
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

```

## ThreadLocal
暂时不纠结源码，反正被他修饰的变量都会为每个线程拷贝一份副本，使得多线程可以无冲突的访问同一个变量且互相不影响。所以这个就保证了上面的prepare方法只会被调用1次，以及每个线程有且只有一个Looper。

## 获取消息
```java
 public static void loop() {
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;
        for (;;) {
            Message msg = queue.next(); // might block
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
         }
    }

```
不断的循环，调用消息队列的next方法取出消息，再调用`msg.target.dispatchMessage`，即Handler的dispatchMessage。

# Handler
```java
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
## CallBack

```java
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
没啥用，在处理的时候发现如果有Callback就调用Callback的handleMessage，就是不想写个类来实现自己的Handler的话使用这个回调来处理Message。

## 发送
```java
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
        private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
`sendMessage`主要设置了msg.target=当前的handler 以及调用MessageQueue来enqueue消息，后续会被Looper获取。

```java
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

```

而`post`是将传入的参数r作为新建的Message的成员callback，在后续的处理中会优先被调用到。

## 处理
```java
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }


    private static void handleCallback(Message message) {
        message.callback.run();
    }
```
在这里先判断msg.callback是否为null，如果我们调用的Handler.post方法，那么这里就会执行`handleCallback`，否则再查看Handler.Callback是否为null，不为null则通过CallBack来处理本次事件，否则调用我们重写的handleMessage。

## HandlerThread

```java
//创建子线程
 mCheckMsgThread = new HandlerThread("check-message-coming");
 mCheckMsgThread.start(); 
//获取Handler
 mCheckMsgHandler = new Handler(mCheckMsgThread.getLooper())
        {
            @Override
            public void handleMessage(Message msg)
            {
				//网络请求操作
            }
        };
//这样就能在主线程中根据UI的生命周期来控制HandlerThread了
 mCheckMsgHandler.sendEmptyMessage(MSG_UPDATE_INFO);//onResume
 mCheckMsgHandler.removeMessages(MSG_UPDATE_INFO);//onPause
```



# 内存泄露
**[update:2018-02-07 10:36:06 ------------------------------------------------------]**

不是说匿名内部类持有外部引用就会导致内存泄漏，而是因为他们的生命周期不同步。 
message.target持有Handler，而Handler又持有Activity引用。 CallBack也一样，是Handler的成员，所以之前说的使用CallBack的方法根本不行，实际测试也是会导致内存泄露。

![](http://owu391pls.bkt.clouddn.com/handlerleak.jpg)

**[update:2018-02-07 10:36:06 ------------------------------------------------------]**

通常我们在Activity中写Handler的时候都是如下的形式：

```java
Handler handler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
```
ide这个时候会报一个内存泄露的警告，为什么呢？

上面说过message.target就是发送该消息的Handler，而上述的Handler是Activity的一个匿名类，java中匿名类和non-static内部类会持有外部类的引用，即Activity。假设Activity被关闭了(**注意区分Activity生命周期和主线程的生命周期**)，但是MessageQueue中的消息还没被处理，那么消息队列中的message就会持有这个Handler的引用，Handler又持有Activity的引用，导致Activity无法被gc回收，导致内存泄漏。

~~通过CallBack，来避免泄露的写法：~~  **这是错误的，正确用法是通过弱引用，详见参考**

```java
Handler handler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            return false;
       }
    });
```

通过静态内部类，也是官方推荐的：

```java
 private static class MyHandler extends Handler {
    private final WeakReference<SampleActivity> mActivity;

    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<SampleActivity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }

  private final MyHandler mHandler = new MyHandler(this);
```



# 参考

[ThreadLocal](http://blog.csdn.net/lufeng20/article/details/24314381)

[官方对Handler泄露的处理](https://www.androiddesignpatterns.com/2013/01/inner-class-handler-memory-leak.html)

[弱引用封装CallBack](https://www.jianshu.com/p/88cf7a923b56)

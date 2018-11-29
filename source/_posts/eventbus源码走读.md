title: EventBus 源码分析
date: 2018-03-18 15:21:11
tags: [事件总线]  
categories: Android框架

事件总线框架EventBus源码走读分析

<!-- more -->  



# 基本使用

在onCreated(init在BaseActivity中的onCreated调用)中注册，onDestroy中取消注册

通过`@Subscribe`注解来修饰订阅事件的处理函数

```java
public class EventBusActivity extends BaseActivity {
    @Override
    public int getLayoutID() {
        return R.layout.activity_eventbus;
    }

    @Override
    public void init() {
        super.init();
        EventBus.getDefault().register(this);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onHandleMyEvent(MyEvent event){
        XLog.i(event.getMessage());
        msgTv.setText(event.getMessage());

    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
    }
}
//自定义的事件类
public class MyEvent {
    private String message;

    public MyEvent(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

在另一个Activity中 通过

```java
 EventBus.getDefault().post(new MyEvent("this msg from TestActivity"));
```

即可发送事件到EventBusActivity

# 源码分析

## register

```java
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
  	//查找缓存
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

  	//ignoreGeneratedIndex 默认为false
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
          
        } else {
          //放到缓存里
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }

private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            //最简单的情况下info==null
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //所以还是会调用这个方法
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

```java
//在findUsingReflectionInSingleClass中就是通过了反射来获取的这个订阅类上所有的方法注解来判断 
//并添加到findState.subscriberMethods中
if (findState.checkAdd(method, eventType)) {
    findState.subscriberMethods.add(
        new SubscriberMethod(method, eventType, threadMode
           ,subscribeAnnotation.priority(),subscribeAnnotation.sticky())   );
}

//findState也是有订阅类来init的
findState.initForSubscriber(subscriberClass);
void initForSubscriber(Class<?> subscriberClass) {
    this.subscriberClass = clazz = subscriberClass;
    skipSuperClasses = false;
    subscriberInfo = null;
}
```

## post

```java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
 			//默认false
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
 		//...
    }
}
private boolean postSingleEventForEventType(/*...*/) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        //在订阅的时候会把 事件类型与对应的Subs添加到这个map中
        //subscriptionsByEventType.put(eventType, subscriptions);
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            //for循环  invoke每个注册登记的方法
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                //...
            }
        }
        return true;
    }
    return false;
}
//来到重头戏
private void postToSubscription(/**/) {
    switch (subscription.subscriberMethod.threadMode) {
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
    }
}
//反射调用  莫得感情 只分析简单的使用的话流程就这么简单
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    }  
}

```

# 参考

[EventBus 3.0进阶：源码及其设计模式 完全解析](https://www.jianshu.com/p/bda4ed3017ba)
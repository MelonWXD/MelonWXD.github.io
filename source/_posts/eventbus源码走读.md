---
title: EventBus 源码分析
date: 2018-03-18 15:21:11
tags: [事件总线]  
categories: Android框架
---

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

## 注册

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
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

最后在`findUsingInfo`中通过`findUsingReflectionInSingleClass(findState);`反射来获取类中被`Subscribe`修饰的方法。



# 参考

[EventBus 3.0进阶：源码及其设计模式 完全解析](https://www.jianshu.com/p/bda4ed3017ba)
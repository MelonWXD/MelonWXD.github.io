---
title: 浅析Android进程间通信（三）
date: 2017-11-11 17:28:11
tags: IPC
categories: Android
---
基于Android 6.0，从源码角度来理解Binder机制
<!-- more -->

# Binder

正如前面说的，进程间通信的本质就是进程A拿到了进程B中Binder的对象的代理，通过这个代理Binder来向进程B发送请求进行通信。而Android系统有各种各样的系统服务，除了上一节提到的系统电量PowerManagerService，还有网络相关的、输入法、音频视频、剪切板和USB等等，想都不用向，这些服务肯定是共同持有一个Binder的抽象类，再具体扩展的。

## BnInterface和BpInterface

先介绍2个来自[IInterface.h](http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IInterface.h#49)的重要接口BnInterface和BpInterface，分别对应`Binder实体`即响应方持有的Binder和`Binder代理`即调用方持有。

```cpp
49  template<typename INTERFACE>
50  class BnInterface : public INTERFACE, public BBinder
51  {
52  public:
53      virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
54      virtual const String16&     getInterfaceDescriptor() const;
55
56  protected:
57      virtual IBinder*            onAsBinder();
58  };
59
60  // ----------------------------------------------------------------------
61
62  template<typename INTERFACE>
63  class BpInterface : public INTERFACE, public BpRefBase
64  {
65  public:
66                                BpInterface(const sp<IBinder>& remote);
67
68  protected:
69      virtual IBinder*            onAsBinder();
70  };
```

BnInterface和BpInterface又分别继承自[Binder.h](http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/Binder.h)中定义的2个类`BBinder`和`BpRefBase`（实际上是和BpRefBase中的mRemote对象BpBinder有关），BBinder和BpBinder都继承自[IBinder](http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IBinder.h)。

关系比较乱，上一个红茶大佬的图：

![](http://img.blog.csdn.net/20150909221454433)





## BBinder

在[Binder.cpp](http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Binder.cpp#73)中对BBinder的各个方法都进行了实现，主要关注`transact`这个方法中调用的`onTransact`，`onTransact`是会被子类重写来实现自己的业务的

```cpp
97 status_t BBinder::transact(
98    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
99 {
100    data.setDataPosition(0);
101
102    status_t err = NO_ERROR;
103    switch (code) {
104        case PING_TRANSACTION:
105            reply->writeInt32(pingBinder());
106            break;
107        default:
108            err = onTransact(code, data, reply, flags);
109            break;
110    }
111
112    if (reply != NULL) {
113        reply->setDataPosition(0);
114    }
115
116    return err;
117 }
```

## BpBinder

[BpBinder](http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/BpBinder.cpp#159)的transact的内部实际调用的是[IPCThreadState::transact](http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp#548)，再跟进就是`IPCThreadState::writeTransactionData`

```cpp
159 status_t BpBinder::transact(
160    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
161{
162    // Once a binder has died, it will never come back to life.
163    if (mAlive) {
164        status_t status = IPCThreadState::self()->transact(
165            mHandle, code, data, reply, flags);
166        if (status == DEAD_OBJECT) mAlive = 0;
167        return status;
168    }
169
170    return DEAD_OBJECT;
171}
```



## 宏

再介绍[IInterface.h](http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IInterface.h)头文件下的几个重要的宏。
### DECLARE_META_INTERFACE
按字面意思直译的话就是`声明interface`，实际上下面的宏也确实是起到了声明的作用
```c
74#define DECLARE_META_INTERFACE(INTERFACE)                               \
75    static const android::String16 descriptor;                          \
76    static android::sp<I##INTERFACE> asInterface(                       \
77            const android::sp<android::IBinder>& obj);                  \
78    virtual const android::String16& getInterfaceDescriptor() const;    \
79    I##INTERFACE();                                                     \
80    virtual ~I##INTERFACE();                 
```
### IMPLEMENT_META_INTERFACE
对应上面的声明，这里就是`实现interface`了。
```c
83#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
84    const android::String16 I##INTERFACE::descriptor(NAME);             \
85    const android::String16&                                            \
86            I##INTERFACE::getInterfaceDescriptor() const {              \
87        return I##INTERFACE::descriptor;                                \
88    }                                                                   \
89    android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
90            const android::sp<android::IBinder>& obj)                   \
91    {                                                                   \
92        android::sp<I##INTERFACE> intr;                                 \
93        if (obj != NULL) {                                              \
94            intr = static_cast<I##INTERFACE*>(                          \
95                obj->queryLocalInterface(                               \
96                        I##INTERFACE::descriptor).get());               \
97            if (intr == NULL) {                                         \
98                intr = new Bp##INTERFACE(obj);                          \
99            }                                                           \
100        }                                                               \
101        return intr;                                                    \
102    }                                                                   \
103    I##INTERFACE::I##INTERFACE() { }                                    \
104    I##INTERFACE::~I##INTERFACE() { }      
```

## 实例分析

下面就以Camera服务来实际理一遍Bn和Bp是如何工作的。

在[Camera.cpp](http://androidxref.com/6.0.1_r10/xref/frameworks/av/camera/Camera.cpp)中各个方法内部实际调用[ICamera.cpp](http://androidxref.com/6.0.1_r10/xref/frameworks/av/camera/ICamera.cpp)来实现

[BnCamera](http://androidxref.com/6.0.1_r10/xref/frameworks/av/include/camera/ICamera.h)在`ICamera.h`中定义了

```java
118class BnCamera: public BnInterface<ICamera>
119{
120public:
121    virtual status_t    onTransact( uint32_t code,
122                                    const Parcel& data,
123                                    Parcel* reply,
124                                    uint32_t flags = 0);
125};
```

而[BpCamera](http://androidxref.com/6.0.1_r10/xref/frameworks/av/camera/ICamera.cpp#54)是ICamera.cpp中定义的类

```cpp
54 class BpCamera: public BpInterface<ICamera>
55 {
56 public:
57    BpCamera(const sp<IBinder>& impl)
58        : BpInterface<ICamera>(impl)
59    {
60    }
	...
	}
```
其他进程请求Camera服务的时候，就需要通过BpCamera来了。下面从BpCamera中一次实际的请求`takePicture`切入分析：
```java
206    // take a picture - returns an IMemory (ref-counted mmap)
207    status_t takePicture(int msgType)
208    {
209        ALOGV("takePicture: 0x%x", msgType);
210        Parcel data, reply;
211        data.writeInterfaceToken(ICamera::getInterfaceDescriptor());
212        data.writeInt32(msgType);
213        remote()->transact(TAKE_PICTURE, data, &reply);
214        status_t ret = reply.readInt32();
215        return ret;
216    }
  ...
   }
```

```java
373        case TAKE_PICTURE: {
374            ALOGV("TAKE_PICTURE");
375            CHECK_INTERFACE(ICamera, data, reply);
376            int msgType = data.readInt32();
377            reply->writeInt32(takePicture(msgType));
378            return NO_ERROR;
379        } break;
```

通过请求码`TAKE_PICTURE`来最后调用到`(takePicture(msgType));`

BpCamera继承BpInterface，是代理Binder。`takePicture`这个方法中data根据一些传输协议来写入数据，通过`remote`来transact，同时还有`reply`作为响应。





# ServiceManager



## 通过0号Binder来获取SM的代理

### JAVA层

在[ServiceManager.java](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/ServiceManager.java)中通过`ServiceManagerNative`来获取ServiceManager

```java
33    private static IServiceManager getIServiceManager() {
34        if (sServiceManager != null) {
35            return sServiceManager;
36        }
37
38        // Find the service manager
39        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
40        return sServiceManager;
41    }
```

在`ServiceManagerNative`中，返回`IServiceManager`的实现类`ServiceManagerProxy`实例，来获取0号Binder

```java
    static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ServiceManagerProxy(obj);
    }
    public ServiceManagerProxy(IBinder remote) 
	{
    	mRemote = remote;//
	}
```


引用红茶话Binder中2句话
> 1） ServiceManagerProxy就是IServiceManager代理接口；
>
> 2） ServiceManagerNative显得很鸡肋；





### C层

在[IServiceManager.cpp](http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IServiceManager.cpp#33)中line40提供了获取0号Binder的方法

```c
33sp<IServiceManager> defaultServiceManager()
34{
35    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
36
37    {
38        AutoMutex _l(gDefaultServiceManagerLock);
39        while (gDefaultServiceManager == NULL) {
40            gDefaultServiceManager = interface_cast<IServiceManager>(
41                ProcessState::self()->getContextObject(NULL));
42            if (gDefaultServiceManager == NULL)
43                sleep(1);
44        }
45    }
46
47    return gDefaultServiceManager;
48}
```
再进入到[ProcessState](http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp#85)中查看相关代码，`getContextObject`中调用`getStrongProxyForHandle(0)`方法。

```c
85sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
86{
87    return getStrongProxyForHandle(0);
88}

179sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
180{
181    sp<IBinder> result;
182
183    AutoMutex _l(mLock);
184
185    handle_entry* e = lookupHandleLocked(handle);
186
187    if (e != NULL) {
191        IBinder* b = e->binder;
192        if (b == NULL || !e->refs->attemptIncWeak(this)) {
193            if (handle == 0) {
212
213                Parcel data;
214                status_t status = IPCThreadState::self()->transact(
215                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
216                if (status == DEAD_OBJECT)
217                   return NULL;
218            }
220            b = new BpBinder(handle);
221            e->binder = b;
222            if (b) e->refs = b->getWeakRefs();
223            result = b;
224        } else {
225            // This little bit of nastyness is to allow us to add a primary
226            // reference to the remote proxy when this team doesn't have one
227            // but another team is sending the handle to us.
228            result.force_set(b);
229            e->refs->decWeak(this);
230        }
231    }
232
233    return result;
234}
```

getContextObject(NULL)实际上相当于返回了一个 new BpBinder(0)

再来看看模板方法`interface_cast`

```c
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```
实际上是调用IServiceManager的asInterface方法



![](http://img.blog.csdn.net/20150909225436079)
## 通过SM代理来向SM注册其他系统服务
```cpp
    power = new PowerManagerService();
    ServiceManager.addService(Context.POWER_SERVICE, power);
```
SystemServer向SM注册PowerManagerService
## 通过SM代理向SM获得其他系统服务

`IServiceManager`的实现类`ServiceManagerProxy`实例中提供了`add`、`get`等方法

```java
public IBinder getService(String name) throws RemoteException 
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    
    mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
    
    IBinder binder = reply.readStrongBinder();
    reply.recycle();
    data.recycle();
    return binder;
}
```

# 参考
[红茶话Binder-1](http://blog.csdn.net/codefly/article/details/17058607)

[Android系统服务](http://blog.csdn.net/sinat_34396176/article/details/51480936)

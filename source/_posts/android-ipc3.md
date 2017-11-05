---
title: android-ipc3
date: 2017-11-03 14:28:11
tags:
categories:
---
在这写文章预览时显示的内容  
<!-- more -->

# ServiceManager
## 通过0号Binder来获取SM的代理
```
gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
                
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```
getContextObject(NULL)实际上相当于new BpBinder(0)
interface_cast实际上是调用IServiceManager的asInterface方法

![](http://img.blog.csdn.net/20150909225436079)
## 通过SM代理来向SM注册其他系统服务
```
    power = new PowerManagerService();
    ServiceManager.addService(Context.POWER_SERVICE, power);
```
SystemServer向SM注册PowerManagerService
## 通过SM代理向SM获得其他系统服务
```
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

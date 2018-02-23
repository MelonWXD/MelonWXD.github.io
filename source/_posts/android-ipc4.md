---
title: 浅析Android进程间通信（四）
date: 2018-02-23 10:13:28
tags: IPC
categories: Android
---

从AIDL的角度来看Binder的实现过程，结合[浅析（一）](https://melonwxd.github.io/2017/10/19/android-ipc/)和[Activity启动流程](https://melonwxd.github.io/2018/01/28/Activity%E5%90%AF%E5%8A%A8%E5%88%86%E6%9E%90/)的讲解来分析。
<!-- more -->

# 分析

```java
public interface MusicAIDLService extends android.os.IInterface {
    
  public static abstract class Stub extends android.os.Binder implements 			       		       com.dongua.ipc.service.MusicAIDLService {
        private static final java.lang.String DESCRIPTOR = "com.dongua.ipc.service.MusicAIDLService";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
    
		public static com.dongua.ipc.service.MusicAIDLService asInterface(android.os.IBinder obj) {
            //省略
        }
    
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }
    
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
           switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_play: {
                    data.enforceInterface(DESCRIPTOR);
                    this.play();
                    reply.writeNoException();
                    return true;
                }
          	//省略
           }
        }


        private static class Proxy implements com.dongua.ipc.service.MusicAIDLService {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            @Override
            public void play() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_play, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_play = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_pause = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_stop = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
        static final int TRANSACTION_prev = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
        static final int TRANSACTION_next = (android.os.IBinder.FIRST_CALL_TRANSACTION + 4);
        static final int TRANSACTION_getCurrentTrack = (android.os.IBinder.FIRST_CALL_TRANSACTION + 5);
    }

    public void play() throws android.os.RemoteException;

    public void pause() throws android.os.RemoteException;

    public void stop() throws android.os.RemoteException;

    public void prev() throws android.os.RemoteException;

    public void next() throws android.os.RemoteException;

    public com.dongua.ipc.service.MusicTrack getCurrentTrack() throws android.os.RemoteException;
}
```

省略了内部类Stub和Proxy的很多实现，先来梳理一下大致的逻辑框架。

- 接口MusicAIDLService，定义了我们在aidl中声明的远程服务方法（play、pause..），并且继承自IInterface
- Stub类继承自Binder，实现上面的MusicAIDLService接口，重写了传输方法onTransact
- Proxy类实现了MusicAIDLService接口，持有IBinder实例mRemote，重写了MusicAIDLService中定义的远程服务方法（play、pause..）

那么我们是怎么使用这些来完成一次跨进程的通信呢

```java
	private MusicAIDLService mMusicServiceProxy ;
    private ServiceConnection mConnection = new ServiceConnection(){
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mMusicServiceProxy = (MusicAIDLService) MusicAIDLService.Stub.asInterface(service);
            try {
                mMusicServiceProxy.play();
                mMusicServiceProxy.pause();
                mMusicServiceProxy.stop();
            } catch (RemoteException e) {
                e.printStackTrace();
            }

        }
    };
```

在onServiceConnected时，通过Stub.asInterface，其实返回的是上述的Proxy类实例，我们通过这个`mMusicServiceProxy`的方法，来调用到远程的服务，如` mMusicServiceProxy.play();`

```java
			@Override
            public void play() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_play, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
```

通过mRemote.transact来传输_data。此时的mRemote就是上面` MusicAIDLService.Stub.asInterface(service)`的service，也就是onServiceConnected方法的第二个参数，即我们通过intent绑定的Service（运行在另一个进程的）的onBind方法的返回值。

由此引入第四个角色，真正的远程服务实现类`mServiceBinder`。

```java
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.i(TAG, "onBind: ");
        return mServiceBinder;
    }

	private Binder mServiceBinder = new MusicAIDLService.Stub() {
        @Override
        public void play() throws RemoteException {
            Log.i(TAG, "play: ");
        }

        @Override
        public void pause() throws RemoteException {
            Log.i(TAG, "pause: ");

        }

        @Override
        public void stop() throws RemoteException {
            Log.i(TAG, "stop: ");
        }

        @Override
        public void prev() throws RemoteException {
            Log.i(TAG, "prev: ");
        }

        @Override
        public void next() throws RemoteException {
            Log.i(TAG, "next: ");
        }

        @Override
        public MusicTrack getCurrentTrack() throws RemoteException {
            Log.i(TAG, "getCurrentTrack: ");
            return null;
        }
    };

```

OK，流程就这样。我把上面总结的重新贴一下：

- MusicAIDLService 接口，定义了我们在aidl中声明的远程服务方法（play、pause等），并且继承自IInterface
- Stub 抽象类，继承自Binder，实现上面的MusicAIDLService接口但没有重写远程服务方法，重写了传输方法`onTransact`，在跨进程时`asInterface`方法返回内部类`Stub.Proxy(obj)`实例
- Proxy 实现了MusicAIDLService接口，持有IBinder实例mRemote，重写了MusicAIDLService中定义的远程服务方法（play、pause等），在其中调用`mRemote.transact()`来进行数据传输
- mServiceBinder 真正的远程音乐服务，业务逻辑实现。继承了Stub，重写了远程服务方法（play、pause等）。



再回过头分析一下ActivityManagerService、ActivityManagerNative、IActivityManager、ActivityManagerProxy四个兄弟。

- IActivityManager 接口，定义了Activity远程服务的方法（startActivity等），并且继承自IInterface
- ActivityManagerNative 抽象类，继承自Binder，实现上面的IActivityManager接口但没有重写远程服务方法，重写了传输方法`onTransact`，`asInterface`返回`ActivityManagerProxy(obj)`实例
- ActivityManagerProxy 实现了IActivityManager接口，持有IBinder实例mRemote，重写了IActivityManager中定义的远程服务方法（startActivity等），在其中调用`mRemote.transact()`来进行数据传输
- ActivityManagerService 真正的远程Activity管理服务类，继承了ActivityManagerNative，重写了远程服务的方法（startActivity等）



关系很明了，无需多言了！


---
title: 浅析Android进程间通信
date: 2017-10-19 20:35:38
tags: [IPC]
categories: Android
---
浅析Android进程间通信
<!-- more -->
## 序列化接口Parcelable
### 为什么会有序列化这么个东西？   
简单说就是为了保存在内存中的各种对象的状态，并且可以把保存的对象状态再读出来。  
虽然你可以用你自己的各种各样的方法来保存，但是Java已经提供一个完善的接口来进行序列化的工作,那就是Serializable。 

### 序列化的使用场景

1. 永久性保存对象，保存对象的字节序列到本地文件中；
2. 通过序列化对象在网络中传递对象；
3. 通过序列化在进程间传递对象。

### Parcelable又是什么？  
Parcelable是Android特有功能，效率比实现Serializable接口高效，可用于Intent数据传递，也可以用于进程间通信（IPC） 
### Parcelable和Serializable如何选择？
 1. 在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable。  
 2. Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC。  
 3. Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点，但此时还是建议使用Serializable 。

> 注意：writeToParcel和参数为Parcel的构造方法，里面的读写顺序一定要一致  

## Android 进程间通信(IPC)
假设要做一个音乐播放的App，除去前台UI展示，还要能够后台播放，实现方案分析可以查看这篇[博文](http://blog.csdn.net/seu_calvin/article/details/53932171)，讲的很深入了。
我们就采用多进程的方式来实现这个需求吧。

### 项目结构：

![](http://owu391pls.bkt.clouddn.com/ss20171016210148.png)
![](http://owu391pls.bkt.clouddn.com/2017-10-24%2022-00-27%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)  

Project视图下可能看的更清楚，在AS中new直接选择AIDL-AIDL File即可自动生成。
- MediaService就是跑在另一个进程中的后台服务，负责播放音乐文件。
- MusicTrack是bean类。
- 要想在AIDL使用到bean类，需要生命与java文件对应的同名aidl文件，即 MusicTrack.aidl
- MusicAIDLService.aidl 就是远程服务接口声明。

> 注意同名的java文件和aild文件的包名需要一样。

### 代码
```java
//MusicAIDLService.aidl 声明远程进程方法 在Service重载
package com.dongua.ipc.service;
import com.dongua.ipc.service.MusicTrack;

interface MusicAIDLService{
    void play();
    void pause();
    void stop();
    void prev();
    void next();
    MusicTrack getCurrentTrack();
}
```

```java
//MusicTrack.aidl 
package com.dongua.ipc.service;
parcelable MusicTrack;
```
```java
//MusicTrack.java 用来多进程之间传输的数据需要实现Parcelable接口
public class MusicTrack implements Parcelable {

    public String mTitle;
    public String mAlbum;
    public String mArtist;

    public MusicTrack(Parcel in) {
        mTitle = in.readString();
        mArtist = in.readString();
        mAlbum = in.readString();
    }

    public static final Creator<MusicTrack> CREATOR = new Creator<MusicTrack>() {
        @Override
        public MusicTrack createFromParcel(Parcel in) {
            return new MusicTrack(in);
        }

        @Override
        public MusicTrack[] newArray(int size) {
            return new MusicTrack[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(mTitle);
        dest.writeString(mAlbum);
        dest.writeString(mArtist);
    }
}
```
 
```java
public class MediaService extends Service {

    private static final String TAG = "MediaService";

    private Binder mBinder = new MusicAIDLService.Stub() {
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




    @Override
    public void onCreate() {
        Log.i(TAG, "onCreate: ");
        super.onCreate();
    }

    @Override
    public void onDestroy() {
        Log.i(TAG, "onDestroy: ");
        super.onDestroy();
    }

    @Override
    public int onStartCommand(Intent intent,  int flags, int startId) {
        Log.i(TAG, "onStartCommand: ");
        return super.onStartCommand(intent, flags, startId);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.i(TAG, "onBind: ");
        return mBinder;
    }


}

```


输出  

```
10-16 20:58:48.257 22211-22211/com.dongua.ipc:music I/MediaService: onCreate: 
10-16 20:58:48.257 22211-22211/com.dongua.ipc:music I/MediaService: onBind: 
10-16 20:58:48.261 22211-22268/com.dongua.ipc:music I/MediaService: play: 
10-16 20:58:48.261 22211-22223/com.dongua.ipc:music I/MediaService: pause: 
10-16 20:58:48.262 22211-22225/com.dongua.ipc:music I/MediaService: stop: 
10-16 20:58:51.633 22211-22211/com.dongua.ipc:music I/MediaService: onDestroy: 
```





## 参考
[深入分析Java的序列化与反序列化](http://www.importnew.com/18024.html)  
[序列化原理机制浅谈](http://blog.csdn.net/morethinkmoretry/article/details/5929345#comments)

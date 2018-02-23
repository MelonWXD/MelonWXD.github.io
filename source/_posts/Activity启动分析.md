---
title: 重学四大组件-Activity启动分析
date: 2018-01-28 19:11:53 
tags: [Activity,源码分析] 
categories: Android  
---

通过6.0源码学习Activity启动的整体流程
<!-- more -->

多嘴一句  本文件创建于`2017-12-01 10:04:40 `

完成于`2018-01-28 19:11:46`

哭了



# 相关知识

在学习整体流程之前先介绍几个和Activity关系比较密切的类。

## SystemServer

之前分析的zygote进程的启动流程，最后是启动了`app_process`这个进程，那么启动之后是怎么样的呢，贴张老罗的图：

![](http://hi.csdn.net/attachment/201109/16/0_1316190384ZuU0.gif)

具体源码流程就略过，最后是启动了[SystemServer](http://androidxref.com/6.0.0_r1/xref/frameworks/base/services/java/com/android/server/SystemServer.java)这个进程，`main`方法如下：

```java
167    public static void main(String[] args) {
168        new SystemServer().run();
169    }

176    private void run() {

246        // Prepare the main looper thread (this thread).
247        android.os.Process.setThreadPriority(
248                android.os.Process.THREAD_PRIORITY_FOREGROUND);
249        android.os.Process.setCanSelfBackground(false);
250        Looper.prepareMainLooper();

258
259        // Initialize the system context.
260        createSystemContext();
261
262        // Create the system service manager.
263        mSystemServiceManager = new SystemServiceManager(mSystemContext);
264        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
  
266        // Start services.
267        try {
268            startBootstrapServices();
269            startCoreServices();
270            startOtherServices();
271        } catch (Throwable ex) {
274            throw ex;
275        }

282        // Loop forever.
283        Looper.loop();
285    }
```

- startBootstrapServices() 启动一些重量级服务 如AMS
- startCoreServices() 启动一些核心服务 如电池管理
- startOtherServices() 启动一些有的没的杂七杂八服务

在`startOtherServices`中，最后会调用

```java
1096        mActivityManagerService.systemReady(new Runnable() {...}
```

在AMS中又调用`startHomeActivityLocked`来启动Launcher组件（桌面）

```java
11719    public void systemReady(final Runnable goingCallback) {
  ...
11876            // Start up initial activity.
11877            mBooting = true;
11878            startHomeActivityLocked(mCurrentUserId, "systemReady");
  ...
}
```

至于`startHomeActivityLocked`怎么搞的，相信看完这篇文章心里大概就有数了。

## ActivityManagerService(AMS)

上面废话说了一堆，还是没说AMS是啥，直译就是Activity管理服务，实际上也是起到这么一个功能。

**划重点了，要考的**

在Android的跨进程通信中，都有这么一个命名套路。有兴趣的可以看看源码，很容易理清楚。

- xxxNative：一般为抽象类，如[ActivityManagerNative](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ActivityManagerNative.java#61)，继承自IActivityManager。实现了一些Binder服务端的一些方法，类似于base的功能吧。
- xxxService：如[ActivityManagerService](http://androidxref.com/6.0.1_r10/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java)，继承自ActivityManagerNative。实现了IActivityManager中声明的所有这个Binder远程服务提供的方法，是远程服务实现类。一些默认实现也会写在AMNative中。
- xxxProxy：如ActivityManagerProxy，为AMNative内部类，通常是外部xxxNative的`getDefault`方法返回值。这个就是AIDL中对应的Proxy类，是远程服务在本地进程的代理，在本地进程我们都是通过xxxNative.getDefault，然后执行某些特定的方法来调用到远程的服务。

这个AMS、AMN、AMP在下面的分析经常会见到，同理ApplicationThreadNative、ApplicationThread(没有Service。。)和ApplicationThreadProxy。

**可以参考[从ActivityManagerNative看Android系统AIDL的实现](https://www.jianshu.com/p/18517a4ef8e1)**







## ActivityThread

ActivityThread代表了一个安卓应用的进程。

Android应用的入口是Application的`onCreate`吗？no！是ActivityThread的`main`，这么一回答，B格是不是就上去了。


再看一下ActivityThread这个类的几个相关的成员变量：

```java
final ApplicationThread mAppThread = new ApplicationThread();
Instrumentation mInstrumentation;//在attach中实例化 line5276
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();//保存了当前App进程所有的Activity，封装成ActivityClientRecord再保存到ArrayMap中
final H mH = new H(); //这个H继承自Handler 定义了许多命令 详见line1227
1227    private class H extends Handler {...}
```

主要关注一下ApplicationThread，后续流程AMS会通过ApplicationThreadProxy，来调用ApplicationThread的服务，从而调用到ApplicationThread外部类，ActivityThread的相关方法来达到控制Activity生命周期的功能。

**ApplicationThread**

```java
574    private class ApplicationThread extends ApplicationThreadNative {...}

//http://androidxref.com/6.0.0_r1/xref/frameworks/base/core/java/android/app/ApplicationThreadNative.java
public abstract class ApplicationThreadNative extends Binder
	       implements IApplicationThread {...}
```





## ActivityStack和ActivityStackSupervisor：

每一个ActivityRecord都会有一个Activity与之对应，一个Activity可能会有多个ActivityRecord，因为Activity可以被多次实例化，取决于其launchmode。一系列相关的ActivityRecord组成了一个TaskRecord，TaskRecord是存在于ActivityStack中，ActivityStackSupervisor是用来管理这些ActivityStack的。

ActivityStack和ActivityRecord的关系如下：
![](http://img.blog.csdn.net/20161220142454652?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2ViZWx6YzI0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**先说个关于ProcessRecord的知识：**

 在AMS中每个ProcessRecord代表一个应用程序进程，AMS的成员ActivityStackSupervisor(ASS)在`startActivityMayWait`中通过caller（IApplicationThread）这个参数，向他的成员mService（指向AMS）调用`getRecordForAppLocked`来获取caller对应的ProcessRecord，包含了对应进程的信息pid、uid等。

 ActivityRecord  TaskRecord  ProcessRecord三者关系图

![](http://img.blog.csdn.net/20170519150259246?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbXdxMzg0ODA3Njgz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)







# Launcher启动Activity过程概述

## Launcher——Activity——Instrumentation——AMP

Launcher逐级调用到Instrumentation，利用`ActivityManagerNative.getDefault()`得到ActivityManagerProxy（AMS的Binder代理）并调用其`startActivity`方法。

## AMP——AMS——ActivityStack

AMP通过Binder来与AMS通信，调用AMS的成员ActivityStackSupervisor(ActivityStackSupervisor内部持有一个ActivityStack)



在ActivityStack中有TaskRecord和ActivityRecord来记录已经启动的Activity的信息

```java
154    /**
155     * The back history of all previous (and possibly still
156     * running) activities.  It contains #TaskRecord objects.
157     */
158    private ArrayList<TaskRecord> mTaskHistory = new ArrayList<>();
```





AMS调用ActivityStack来执行启动Activity操作，先使用`PackageManagerService`去解析启动参数Intent中各种信息(判断启动flag啊，获取caller的ProcessRecord进程信息，是否需要新建一个ProcessRecord等等)。之后再根据Intent的信息(newTask)来判断是否要把即将启动的Activity放到上述的ActivityStack成员mTaskHistory的顶部。后续检查一些是否mResumeActivity就是要启动，是否有结束Activity动作发生，关机等等等。都OK的话才会通知当前的Activity-mResumeActivity（如果从桌面启动的话就是Launcher组件）来进入onPaused流程。



## ApplicationThreadProxy——ApplicationThread——ActivityThread

接上面的，mResumeActivity的onPaused流程是怎么通知的呢。这样就要说到上述的ActivityRecord的成员app(ProcessRecord)的又一个成员变量thread(ApplicationThreadProxy)，看这个类型就知道是干嘛的，Binder。

插入介绍一下AppTP、AppTN、AppT吧。

ApplicationThread是[ActivityThread](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ActivityThread.java)的内部类，继承自AppTN，其实就跟AMS继承AMN以及AMP代理的关系一样。

```java
   private class ApplicationThread extends ApplicationThreadNative {...}
```

好，继续。AppTP通过Binder通知ApplicationThread处理onPaused，会调用ApplicationThread的外部类ActivityThread的Handler-Message机制来通知ActivityThread，在ActivityThread的处理消息的函数中。

先是让当前的Activity进入onPause

```java
  performPauseActivity(token, finished, r.isPreHoneycomb());
```

又通过这么一行代码来通知AMS说我已经让它pause了

```java
ActivityManagerNative.getDefault().activityPaused(token);//ActivityManagerNative.getDefault()=ActivityManagerProxy 详细见下面代码分析
```

艾玛，太高兴了，又回到了ActivityManagerProxy。

## AMP——AMS——ActivityStack

通知Stack要进入onPause的mResumeActivity已经被暂停了

## ActivityStack——AMS——ActivityThread

mResumeActivity执行完之后，再来启动栈顶Activity。在ActivityStack中检查，PID、UID啥的，是否该Activity所属的进程已经启动了，如果启动了，就通知那个进程来启动Activity，如果没有就以PID和UID新建一个进程，再通知这个新进程启动Activity。

下面假设需要新建ProcessRecord，新建完肯定要保存相应的ProcessRecord到AMS。然后执行ActivityThread的main方法。

**ApplicationThread的main方法就是一个App的入口方法**

## ActivityThread——AMP——AMS——ActivityStack

ActivityThread的main中调用attach，其中通过AMP执行attachApplication，调用到AMS的attachApplication，然后又进入Stack一顿瞎比操作。

跟上面暂停的流程很像，ApplicationThreadProxy通过Binder远程调用ApplicationThread，Handler-Message调用外部类ActivityThread实际函数`performLaunchActivity`，在调用`mInstrumention.callActivityOnCreate`














# Activity的启动

**以下是无脑跟源码过程，可略！**

从Activity的startActivity方法开始一步步跟进，走读Activity的启动流程，子标题为相关类的递进

## Activity->Instrumentation->ActivityManagerProxy



一步步跟进[Activity](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/Activity.java#3927)的`startActivity`方法，最后都是调用的`startActivityForResult`

```java
3927    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
3928        if (mParent == null) {
3929            Instrumentation.ActivityResult ar =
3930                mInstrumentation.execStartActivity(
3931                    this, mMainThread.getApplicationThread(), mToken, this,
3932                    intent, requestCode, options);
3933            if (ar != null) {
3934                mMainThread.sendActivityResult(
3935                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
3936                    ar.getResultData());
3937            }
3938            if (requestCode >= 0) {
3939                // If this start is requesting a result, we can avoid making
3940                // the activity visible until the result is received.  Setting
3941                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
3942                // activity hidden during this time, to avoid flickering.
3943                // This can only be done when a result is requested because
3944                // that guarantees we will get information back when the
3945                // activity is finished, no matter what happens to it.
3946                mStartedActivity = true;
3947            }
3948
3949            cancelInputsAndStartExitTransition(options);
3950            // TODO Consider clearing/flushing other event sources and events for child windows.
3951        } else {
3952            if (options != null) {
3953                mParent.startActivityFromChild(this, intent, requestCode, options);
3954            } else {
3955                // Note we want to go through this method for compatibility with
3956                // existing applications that may have overridden it.
3957                mParent.startActivityFromChild(this, intent, requestCode);
3958            }
3959        }
3960    }
```

实际上是调用的`mInstrumentation.execStartActivity`，跟进

```java
1604    public ActivityResult execStartActivity(
1605        Context who, IBinder contextThread, IBinder token, String target,
1606        Intent intent, int requestCode, Bundle options) {
1607        IApplicationThread whoThread = (IApplicationThread) contextThread;
1608        if (mActivityMonitors != null) {
1609            synchronized (mSync) {
1610                final int N = mActivityMonitors.size();
1611                for (int i=0; i<N; i++) {
 						...//查找Activity是否已经存在
1620                }
1621            }
1622        }
1623        try {
1624            intent.migrateExtraStreamToClipData();
1625            intent.prepareToLeaveProcess();
1626            int result = ActivityManagerNative.getDefault()
1627                .startActivity(whoThread, who.getBasePackageName(), intent,
1628                        intent.resolveTypeIfNeeded(who.getContentResolver()),
1629                        token, target, requestCode, 0, null, options);
1630            checkStartActivityResult(result, intent);
1631        } catch (RemoteException e) {
1632            throw new RuntimeException("Failure from system", e);
1633        }
1634        return null;
1635    }

```

**注意这里的`whoThread`参数，IApplicationThread的实例，在后续会说明**

实际上是在`try`块中调用`ActivityManagerNative.getDefault().startActivity`，那么`ActivityManagerNative.getDefault()`是啥捏，去[ActivityManagerNative(AMN)](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ActivityManagerNative.java)里头看看

```java
2604    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
2605        protected IActivityManager create() {
2606            IBinder b = ServiceManager.getService("activity");
2607            if (false) {
2608                Log.v("ActivityManager", "default service binder = " + b);
2609            }
2610            IActivityManager am = asInterface(b);
2611            if (false) {
2612                Log.v("ActivityManager", "default service = " + am);
2613            }
2614            return am;
2615        }
2616    };
```

就是一个单例的[IActivityManager](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/IActivityManager.java#66) IActivityManager是一个继承IInterface的接口，可以基本确定是负责Binder通信的了，内部的方法都和Activity的生命周期相关。

```java
 public interface IActivityManager extends IInterface {
   ...
     93    public boolean startNextMatchingActivity(IBinder callingActivity,
94            Intent intent, Bundle options) throws RemoteException;
95    public int startActivityFromRecents(int taskId, Bundle options) throws RemoteException;
96    public boolean finishActivity(IBinder token, int code, Intent data, boolean finishTask)
97            throws RemoteException;
98    public void finishSubActivity(IBinder token, String resultWho, int requestCode) throws RemoteException;
99    public boolean finishActivityAffinity(IBinder token) throws RemoteException;
100    public void finishVoiceTask(IVoiceInteractionSession session) throws RemoteException;
101    public boolean releaseActivityInstance(IBinder token) throws RemoteException;
102    public void releaseSomeActivities(IApplicationThread app) throws RemoteException;
103    public boolean willActivityBeVisible(IBinder token) throws RemoteException;
   ...
}
```

那么上面gDefault中返回的`am`到底是那个实现类呢，跟进`IActivityManager am = asInterface(b)`

```java
67    static public IActivityManager asInterface(IBinder obj) {
68        if (obj == null) {
69            return null;
70        }
71        IActivityManager in =
72            (IActivityManager)obj.queryLocalInterface(descriptor);
73        if (in != null) {
74            return in;
75        }
76
77        return new ActivityManagerProxy(obj);
78    }
```

所以` ActivityManagerNative.getDefault()`返回的是`ActivityManagerProxy`，是ActivityManagerNative的一个内部类，他俩都实现了IActivityManager这个接口，重点关注一下构造方法，传入的是一个 **IBinder** 实例

```java
    public ActivityManagerProxy(IBinder remote)  
    {  
        mRemote = remote;  
    }  
```

`跟进查看`startActivity`方法

```java
2631    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
2632            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
2633            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
2634        Parcel data = Parcel.obtain();
2635        Parcel reply = Parcel.obtain();
2636        data.writeInterfaceToken(IActivityManager.descriptor);
2637        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
2638        data.writeString(callingPackage);
2639        intent.writeToParcel(data, 0);
2640        data.writeString(resolvedType);
2641        data.writeStrongBinder(resultTo);
2642        data.writeString(resultWho);
2643        data.writeInt(requestCode);
2644        data.writeInt(startFlags);
2645        if (profilerInfo != null) {
2646            data.writeInt(1);
2647            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
2648        } else {
2649            data.writeInt(0);
2650        }
2651        if (options != null) {
2652            data.writeInt(1);
2653            options.writeToParcel(data, 0);
2654        } else {
2655            data.writeInt(0);
2656        }
2657        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
2658        reply.readException();
2659        int result = reply.readInt();
2660        reply.recycle();
2661        data.recycle();
2662        return result;
2663    }
```

很明了，使用`mRemote`成员进行Binder通信，通信码`START_ACTIVITY_TRANSACTION`。

往上回溯，这个mRemote就是`ServiceManager.getService("activity")`的返回值，那么这个的返回值是啥捏

是`ActivityManagerService`，让我们进入 [ActivityManagerService](http://androidxref.com/6.0.1_r10/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java)的代码来找证据

```java
2172    public void setSystemProcess() {
2173        try {
2174            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);//重点看这 看这！
2175            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
2176            ServiceManager.addService("meminfo", new MemBinder(this));
2177            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
2178            ServiceManager.addService("dbinfo", new DbBinder(this));
2179            if (MONITOR_CPU_USAGE) {
2180                ServiceManager.addService("cpuinfo", new CpuBinder(this));
2181            }
2182            ServiceManager.addService("permission", new PermissionController(this));
2183            ServiceManager.addService("processinfo", new ProcessInfoService(this));
2184
2185            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
2186                    "android", STOCK_PM_FLAGS);
2187            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

2201        } catch (PackageManager.NameNotFoundException e) {
2202            throw new RuntimeException(
2203                    "Unable to find android system package", e);
2204        }
2205    }
```

Context.java中定义了**ACTIVITY_SERVICE**=activity

`public static final String ACTIVITY_SERVICE = "activity" `

ActivityManagerService又把自己注册为`activity`这个服务，所以`ServiceManager.getService("activity")`的返回值就是ActivityManagerService。所以我们就进入`ActivityManagerService`分析一波。

## ActivityManagerService->ActivityStackSupervisor

我们是通过mRmote向AMS进行IPC通信，根据Binder的实现原理，会调用到AMS的`onTransact`方法

```java
2452    @Override
2453    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
2454            throws RemoteException {
2455        if (code == SYSPROPS_TRANSACTION) {
2456            // We need to tell all apps about the system property change.
				...
2481        }
2482        try {
2483            return super.onTransact(code, data, reply, flags);
2484        } catch (RuntimeException e) {
2485            // The activity manager only throws security exceptions, so let's
2486            // log all others.
2487            if (!(e instanceof SecurityException)) {
2488                Slog.wtf(TAG, "Activity Manager Crash", e);
2489            }
2490            throw e;
2491        }
2492    }
```

实际上只响应SYSPROPS_TRANSACTION请求码，其他的通过`super`调用AMS的父类，ActivityManagerNative来处理，那么查看一下ActivityManagerNative的`onTransact`中对应code为`START_ACTIVITY_TRANSACTION`的方法块吧

```java
146        case START_ACTIVITY_TRANSACTION:
147        {
148            data.enforceInterface(IActivityManager.descriptor);
149            IBinder b = data.readStrongBinder();
150            IApplicationThread app = ApplicationThreadNative.asInterface(b);
151            String callingPackage = data.readString();
152            Intent intent = Intent.CREATOR.createFromParcel(data);
153            String resolvedType = data.readString();
154            IBinder resultTo = data.readStrongBinder();
155            String resultWho = data.readString();
156            int requestCode = data.readInt();
157            int startFlags = data.readInt();
158            ProfilerInfo profilerInfo = data.readInt() != 0
159                    ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
160            Bundle options = data.readInt() != 0
161                    ? Bundle.CREATOR.createFromParcel(data) : null;
162            int result = startActivity(app, callingPackage, intent, resolvedType,
163                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
164            reply.writeNoException();
165            reply.writeInt(result);
166            return true;
167        }
```

烦不烦啊，怎么又调用`startActivity`，但是注意了啊，这个`startActivity`是**AMS**中的，而不是上面的ActivityManagerNative中的方法。

```java
3848    @Override
3849    public final int startActivity(IApplicationThread caller, String callingPackage,
3850            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
3851            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
3852        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
3853            resultWho, requestCode, startFlags, profilerInfo, options,
3854            UserHandle.getCallingUserId());
3855    }
3856
3857    @Override
3858    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
3859            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
3860            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
3861        enforceNotIsolatedCaller("startActivity");
3862        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
3863                false, ALLOW_FULL_ONLY, "startActivity", null);
3864        // TODO: Switch to user app stacks here.
3865        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
3866                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
3867                profilerInfo, null, null, options, false, userId, null, null);
3868    }
```

冲啊，跟进代码[mStackSupervisor.startActivityMayWait](http://androidxref.com/6.0.1_r10/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java#925)

```java
 final int startActivityMayWait(IApplicationThread caller, int callingUid,
926            String callingPackage, Intent intent, String resolvedType,
927            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
928            IBinder resultTo, String resultWho, int requestCode, int startFlags,
929            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
930            Bundle options, boolean ignoreTargetSecurity, int userId,
931            IActivityContainer iContainer, TaskRecord inTask) {

//解析Intent信息
938        // Don't modify the client's object!
939        intent = new Intent(intent);
941        // Collect information about the target of the Intent.
942        ActivityInfo aInfo =
943                resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);

946        synchronized (mService) {

1045            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
1046                    voiceSession, voiceInteractor, resultTo, resultWho,
1047                    requestCode, callingPid, callingUid, callingPackage,
1048                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,

1095
1096            return res;
1097        }
1098    }
```

代码太长 删减了许多，有兴趣的自己看源码去，跟进`res = startActivityLocked`

```java
    final int startActivityLocked(IApplicationThread caller,
1400            Intent intent, String resolvedType, ActivityInfo aInfo,
1401            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
1402            IBinder resultTo, String resultWho, int requestCode,
1403            int callingPid, int callingUid, String callingPackage,
1404            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
1405            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
1406            ActivityContainer container, TaskRecord inTask) {
1407        int err = ActivityManager.START_SUCCESS;
//从传进来的参数caller得到调用者的进程信息，并保存在callerApp变量中
1409        ProcessRecord callerApp = null;
1410        if (caller != null) {
1411            callerApp = mService.getRecordForAppLocked(caller);
1412            if (callerApp != null) {
1413                callingPid = callerApp.pid;
1414                callingUid = callerApp.info.uid;
1415            } else {
1416                Slog.w(TAG, "Unable to find app for caller " + caller
1417                      + " (pid=" + callingPid + ") when starting: "
1418                      + intent.toString());
1419                err = ActivityManager.START_PERMISSION_DENIED;
1420            }
1421        }
1422

//创建即将要启动的Activity的相关信息，并保存在r变量
1636        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
1637                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
1638                requestCode, componentSpecified, voiceSession != null, this, container, options);

1674
1675        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
1676                startFlags, true, options, inTask);
1677
1678        if (err < 0) {
1679            // If someone asked to have the keyguard dismissed on the next
1680            // activity start, but we are not actually doing an activity
1681            // switch...  just dismiss the keyguard now, because we
1682            // probably want to see whatever is behind it.
1683            notifyActivityDrawnForKeyguard();
1684        }
1685        return err;
1686    }
```

代码量实在太多了，对一些信息的封装和检测啥的，真不大懂，无脑跟进，go`startActivityUncheckedLocked`

```java
 final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
1829            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
1830            boolean doResume, Bundle options, TaskRecord inTask) {
1831        final Intent intent = r.intent;
1832        final int callingUid = r.launchedFromUid;
1833

1847        int launchFlags = intent.getFlags();
1848        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 &&
1849                (launchSingleInstance || launchSingleTask)) {
1850            // We have a conflict between the Intent and the Activity manifest, manifest wins.
1851            Slog.i(TAG, "Ignoring FLAG_ACTIVITY_NEW_DOCUMENT, launchMode is " +
1852                    "\"singleInstance\" or \"singleTask\"");
1853            launchFlags &=
1854                    ~(Intent.FLAG_ACTIVITY_NEW_DOCUMENT | Intent.FLAG_ACTIVITY_MULTIPLE_TASK);
1855        } else {
1856            switch (r.info.documentLaunchMode) {
1857                case ActivityInfo.DOCUMENT_LAUNCH_NONE:
1858                    break;
1859                case ActivityInfo.DOCUMENT_LAUNCH_INTO_EXISTING:
1860                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
1861                    break;
1862                case ActivityInfo.DOCUMENT_LAUNCH_ALWAYS:
1863                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
1864                    break;
1865                case ActivityInfo.DOCUMENT_LAUNCH_NEVER:
1866                    launchFlags &= ~Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
1867                    break;
1868            }
1869        }
1870
1871        final boolean launchTaskBehind = r.mLaunchTaskBehind
1872                && !launchSingleTask && !launchSingleInstance
1873                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0;
1874

2457        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
2458        targetStack.mLastPausedActivity = null;
2459        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
2464        return ActivityManager.START_SUCCESS;
2465    }
```

一堆的标记位Flag检测，来判断启动行为，看不懂，放弃，go`startActivityLocked`

```java

1399    final int startActivityLocked(IApplicationThread caller,
1400            Intent intent, String resolvedType, ActivityInfo aInfo,
1401            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
1402            IBinder resultTo, String resultWho, int requestCode,
1403            int callingPid, int callingUid, String callingPackage,
1404            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
1405            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
1406            ActivityContainer container, TaskRecord inTask) {
1407        int err = ActivityManager.START_SUCCESS;
1408
1409        ProcessRecord callerApp = null;
1410        if (caller != null) {
1411            callerApp = mService.getRecordForAppLocked(caller);
1412            if (callerApp != null) {
1413                callingPid = callerApp.pid;
1414                callingUid = callerApp.info.uid;
1415            } else {
1416                Slog.w(TAG, "Unable to find app for caller " + caller
1417                      + " (pid=" + callingPid + ") when starting: "
1418                      + intent.toString());
1419                err = ActivityManager.START_PERMISSION_DENIED;
1420            }
1421        }
1422
1423        final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;
1424
1425        if (err == ActivityManager.START_SUCCESS) {
1426            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
1427                    + "} from uid " + callingUid
1428                    + " on display " + (container == null ? (mFocusedStack == null ?
1429                            Display.DEFAULT_DISPLAY : mFocusedStack.mDisplayId) :
1430                            (container.mActivityDisplay == null ? Display.DEFAULT_DISPLAY :
1431                                    container.mActivityDisplay.mDisplayId)));
1432        }
1433
1434        ActivityRecord sourceRecord = null;
1435        ActivityRecord resultRecord = null;
1436        if (resultTo != null) {
1437            sourceRecord = isInAnyStackLocked(resultTo);
1438            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
1439                    "Will send result to " + resultTo + " " + sourceRecord);
1440            if (sourceRecord != null) {
1441                if (requestCode >= 0 && !sourceRecord.finishing) {
1442                    resultRecord = sourceRecord;
1443                }
1444            }
1445        }
1446
1447        final int launchFlags = intent.getFlags();
1448
1449        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
1450            // Transfer the result target from the source activity to the new
1451            // one being started, including any failures.
1452            if (requestCode >= 0) {
1453                ActivityOptions.abort(options);
1454                return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;
1455            }
1456            resultRecord = sourceRecord.resultTo;
1457            if (resultRecord != null && !resultRecord.isInStackLocked()) {
1458                resultRecord = null;
1459            }
1460            resultWho = sourceRecord.resultWho;
1461            requestCode = sourceRecord.requestCode;
1462            sourceRecord.resultTo = null;
1463            if (resultRecord != null) {
1464                resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);
1465            }
1466            if (sourceRecord.launchedFromUid == callingUid) {
1467                // The new activity is being launched from the same uid as the previous
1468                // activity in the flow, and asking to forward its result back to the
1469                // previous.  In this case the activity is serving as a trampoline between
1470                // the two, so we also want to update its launchedFromPackage to be the
1471                // same as the previous activity.  Note that this is safe, since we know
1472                // these two packages come from the same uid; the caller could just as
1473                // well have supplied that same package name itself.  This specifially
1474                // deals with the case of an intent picker/chooser being launched in the app
1475                // flow to redirect to an activity picked by the user, where we want the final
1476                // activity to consider it to have been launched by the previous app activity.
1477                callingPackage = sourceRecord.launchedFromPackage;
1478            }
1479        }
1480
1481        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
1482            // We couldn't find a class that can handle the given Intent.
1483            // That's the end of that!
1484            err = ActivityManager.START_INTENT_NOT_RESOLVED;
1485        }
1486
1487        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
1488            // We couldn't find the specific class specified in the Intent.
1489            // Also the end of the line.
1490            err = ActivityManager.START_CLASS_NOT_FOUND;
1491        }
1492
1493        if (err == ActivityManager.START_SUCCESS
1494                && !isCurrentProfileLocked(userId)
1495                && (aInfo.flags & FLAG_SHOW_FOR_ALL_USERS) == 0) {
1496            // Trying to launch a background activity that doesn't show for all users.
1497            err = ActivityManager.START_NOT_CURRENT_USER_ACTIVITY;
1498        }
1499
1500        if (err == ActivityManager.START_SUCCESS && sourceRecord != null
1501                && sourceRecord.task.voiceSession != null) {
1502            // If this activity is being launched as part of a voice session, we need
1503            // to ensure that it is safe to do so.  If the upcoming activity will also
1504            // be part of the voice session, we can only launch it if it has explicitly
1505            // said it supports the VOICE category, or it is a part of the calling app.
1506            if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
1507                    && sourceRecord.info.applicationInfo.uid != aInfo.applicationInfo.uid) {
1508                try {
1509                    intent.addCategory(Intent.CATEGORY_VOICE);
1510                    if (!AppGlobals.getPackageManager().activitySupportsIntent(
1511                            intent.getComponent(), intent, resolvedType)) {
1512                        Slog.w(TAG,
1513                                "Activity being started in current voice task does not support voice: "
1514                                + intent);
1515                        err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
1516                    }
1517                } catch (RemoteException e) {
1518                    Slog.w(TAG, "Failure checking voice capabilities", e);
1519                    err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
1520                }
1521            }
1522        }
1523
1524        if (err == ActivityManager.START_SUCCESS && voiceSession != null) {
1525            // If the caller is starting a new voice session, just make sure the target
1526            // is actually allowing it to run this way.
1527            try {
1528                if (!AppGlobals.getPackageManager().activitySupportsIntent(intent.getComponent(),
1529                        intent, resolvedType)) {
1530                    Slog.w(TAG,
1531                            "Activity being started in new voice task does not support: "
1532                            + intent);
1533                    err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
1534                }
1535            } catch (RemoteException e) {
1536                Slog.w(TAG, "Failure checking voice capabilities", e);
1537                err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
1538            }
1539        }
1540
1541        final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;
1542
1543        if (err != ActivityManager.START_SUCCESS) {
1544            if (resultRecord != null) {
1545                resultStack.sendActivityResultLocked(-1,
1546                    resultRecord, resultWho, requestCode,
1547                    Activity.RESULT_CANCELED, null);
1548            }
1549            ActivityOptions.abort(options);
1550            return err;
1551        }
1552
1553        boolean abort = false;
1554
1555        final int startAnyPerm = mService.checkPermission(
1556                START_ANY_ACTIVITY, callingPid, callingUid);
1557
1558        if (startAnyPerm != PERMISSION_GRANTED) {
1559            final int componentRestriction = getComponentRestrictionForCallingPackage(
1560                    aInfo, callingPackage, callingPid, callingUid, ignoreTargetSecurity);
1561            final int actionRestriction = getActionRestrictionForCallingPackage(
1562                    intent.getAction(), callingPackage, callingPid, callingUid);
1563
1564            if (componentRestriction == ACTIVITY_RESTRICTION_PERMISSION
1565                    || actionRestriction == ACTIVITY_RESTRICTION_PERMISSION) {
1566                if (resultRecord != null) {
1567                    resultStack.sendActivityResultLocked(-1,
1568                            resultRecord, resultWho, requestCode,
1569                            Activity.RESULT_CANCELED, null);
1570                }
1571                String msg;
1572                if (actionRestriction == ACTIVITY_RESTRICTION_PERMISSION) {
1573                    msg = "Permission Denial: starting " + intent.toString()
1574                            + " from " + callerApp + " (pid=" + callingPid
1575                            + ", uid=" + callingUid + ")" + " with revoked permission "
1576                            + ACTION_TO_RUNTIME_PERMISSION.get(intent.getAction());
1577                } else if (!aInfo.exported) {
1578                    msg = "Permission Denial: starting " + intent.toString()
1579                            + " from " + callerApp + " (pid=" + callingPid
1580                            + ", uid=" + callingUid + ")"
1581                            + " not exported from uid " + aInfo.applicationInfo.uid;
1582                } else {
1583                    msg = "Permission Denial: starting " + intent.toString()
1584                            + " from " + callerApp + " (pid=" + callingPid
1585                            + ", uid=" + callingUid + ")"
1586                            + " requires " + aInfo.permission;
1587                }
1588                Slog.w(TAG, msg);
1589                throw new SecurityException(msg);
1590            }
1591
1592            if (actionRestriction == ACTIVITY_RESTRICTION_APPOP) {
1593                String message = "Appop Denial: starting " + intent.toString()
1594                        + " from " + callerApp + " (pid=" + callingPid
1595                        + ", uid=" + callingUid + ")"
1596                        + " requires " + AppOpsManager.permissionToOp(
1597                                ACTION_TO_RUNTIME_PERMISSION.get(intent.getAction()));
1598                Slog.w(TAG, message);
1599                abort = true;
1600            } else if (componentRestriction == ACTIVITY_RESTRICTION_APPOP) {
1601                String message = "Appop Denial: starting " + intent.toString()
1602                        + " from " + callerApp + " (pid=" + callingPid
1603                        + ", uid=" + callingUid + ")"
1604                        + " requires appop " + AppOpsManager.permissionToOp(aInfo.permission);
1605                Slog.w(TAG, message);
1606                abort = true;
1607            }
1608        }
1609
1610        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
1611                callingPid, resolvedType, aInfo.applicationInfo);
1612
1613        if (mService.mController != null) {
1614            try {
1615                // The Intent we give to the watcher has the extra data
1616                // stripped off, since it can contain private information.
1617                Intent watchIntent = intent.cloneFilter();
1618                abort |= !mService.mController.activityStarting(watchIntent,
1619                        aInfo.applicationInfo.packageName);
1620            } catch (RemoteException e) {
1621                mService.mController = null;
1622            }
1623        }
1624
1625        if (abort) {
1626            if (resultRecord != null) {
1627                resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
1628                        Activity.RESULT_CANCELED, null);
1629            }
1630            // We pretend to the caller that it was really started, but
1631            // they will just get a cancel result.
1632            ActivityOptions.abort(options);
1633            return ActivityManager.START_SUCCESS;
1634        }
1635
1636        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
1637                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
1638                requestCode, componentSpecified, voiceSession != null, this, container, options);
1639        if (outActivity != null) {
1640            outActivity[0] = r;
1641        }
1642
1643        if (r.appTimeTracker == null && sourceRecord != null) {
1644            // If the caller didn't specify an explicit time tracker, we want to continue
1645            // tracking under any it has.
1646            r.appTimeTracker = sourceRecord.appTimeTracker;
1647        }
1648
1649        final ActivityStack stack = mFocusedStack;
1650        if (voiceSession == null && (stack.mResumedActivity == null
1651                || stack.mResumedActivity.info.applicationInfo.uid != callingUid)) {
1652            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
1653                    realCallingPid, realCallingUid, "Activity start")) {
1654                PendingActivityLaunch pal =
1655                        new PendingActivityLaunch(r, sourceRecord, startFlags, stack);
1656                mPendingActivityLaunches.add(pal);
1657                ActivityOptions.abort(options);
1658                return ActivityManager.START_SWITCHES_CANCELED;
1659            }
1660        }
1661
1662        if (mService.mDidAppSwitch) {
1663            // This is the second allowed switch since we stopped switches,
1664            // so now just generally allow switches.  Use case: user presses
1665            // home (switches disabled, switch to home, mDidAppSwitch now true);
1666            // user taps a home icon (coming from home so allowed, we hit here
1667            // and now allow anyone to switch again).
1668            mService.mAppSwitchesAllowedTime = 0;
1669        } else {
1670            mService.mDidAppSwitch = true;
1671        }
1672
1673        doPendingActivityLaunchesLocked(false);
1674
1675        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
1676                startFlags, true, options, inTask);
1677
1678        if (err < 0) {
1679            // If someone asked to have the keyguard dismissed on the next
1680            // activity start, but we are not actually doing an activity
1681            // switch...  just dismiss the keyguard now, because we
1682            // probably want to see whatever is behind it.
1683            notifyActivityDrawnForKeyguard();
1684        }
1685        return err;
1686    }
```



代码都是多的要死，在startActivityLocked内部主要就是把Activity放到Activity组件堆栈中，然后在激活栈顶的Activity来达到启动Activity的目的。

然后`startActivityUncheckedLocked`-> `ActivityStack.resumeTopActivityLocked`来通知Activity栈最上层不为Finish态的Activity进入Paused状态->`resumeTopActivityInnerLocked`->`startPausingLocked`来设置ActivityRecord状态和调用ApplicationThreadProxy的`schedulePauseActivity`方法

#ApplicationThreadProxy->ApplicationThread->ActivityThread

ApplicationThreadProxy是[ApplicationThreadNative](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ApplicationThreadNative.java)的内部类

```java
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
        ...
	
	
	class ApplicationThreadProxy implements IApplicationThread {
        ...  
        }
}
```

跟之前说的ActivityManagerNative和ActivityManagerProxy异曲同工，ApplicationThreadProxy发送的transact会在ActivityManagerNative的子类[ApplicationThread](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ActivityThread.java#574)`onTransact`方法处理，ApplicationThread是ActivityThread的内部类，同时也没有覆盖`onTransact`方法。

```java
// ApplicationThreadProxy.class
718    public final void schedulePauseActivity(IBinder token, boolean finished,
719            boolean userLeaving, int configChanges, boolean dontReport) throws RemoteException {
720        Parcel data = Parcel.obtain();
721        data.writeInterfaceToken(IApplicationThread.descriptor);
722        data.writeStrongBinder(token);
723        data.writeInt(finished ? 1 : 0);
724        data.writeInt(userLeaving ? 1 :0);
725        data.writeInt(configChanges);
726        data.writeInt(dontReport ? 1 : 0);
727        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
728                IBinder.FLAG_ONEWAY);
729        data.recycle();
730    }
731
 // ActivityManagerNative.java
  78        case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
79        {
80            data.enforceInterface(IApplicationThread.descriptor);
81            IBinder b = data.readStrongBinder();
82            boolean finished = data.readInt() != 0;
83            boolean userLeaving = data.readInt() != 0;
84            int configChanges = data.readInt();
85            boolean dontReport = data.readInt() != 0;
86            schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
87            return true;
88        }
//ApplicationThread.class 
588        public final void schedulePauseActivity(IBinder token, boolean finished,
589                boolean userLeaving, int configChanges, boolean dontReport) {
590            sendMessage(
591                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
592                    token,
593                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
594                    configChanges);
595        }

  
```

跟进ActivityThread成员函数`sendMessage`

```java
2265    private void sendMessage(int what, Object obj, int arg1, int arg2) {
2266        sendMessage(what, obj, arg1, arg2, false);
2267    }
2268
2269    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
2270        if (DEBUG_MESSAGES) Slog.v(
2271            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
2272            + ": " + arg1 + " / " + obj);
2273        Message msg = Message.obtain();
2274        msg.what = what;
2275        msg.obj = obj;
2276        msg.arg1 = arg1;
2277        msg.arg2 = arg2;
2278        if (async) {
2279            msg.setAsynchronous(true);
2280        }
2281        mH.sendMessage(msg);
2282    }
```

这就是handler-message机制了，注意这个what字段值为**PAUSE_ACTIVITY**，跟进`handleMessage`方法！！！

```java
1353                case PAUSE_ACTIVITY:
1354                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
1355                    handlePauseActivity((IBinder)msg.obj, false, (msg.arg1&1) != 0, msg.arg2,
1356                            (msg.arg1&2) != 0);
1357                    maybeSnapshot();
1358                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
1359                    break;
```

```java
3305    private void handlePauseActivity(IBinder token, boolean finished,
3306            boolean userLeaving, int configChanges, boolean dontReport) {
3307        ActivityClientRecord r = mActivities.get(token);
3308        if (r != null) {

3322            // Tell the activity manager we have paused.
3323            if (!dontReport) {
3324                try {
3325                    ActivityManagerNative.getDefault().activityPaused(token);
3326                } catch (RemoteException ex) {
3327                }
3328            }
3329            mSomeActivitiesChanged = true;
3330        }
3331    }
```

之前说过了ActivityManagerNative.getDefault()=ActivityManagerProxy，所以就跟进

```java
3134    public void activityPaused(IBinder token) throws RemoteException
3135    {
3136        Parcel data = Parcel.obtain();
3137        Parcel reply = Parcel.obtain();
3138        data.writeInterfaceToken(IActivityManager.descriptor);
3139        data.writeStrongBinder(token);
3140        mRemote.transact(ACTIVITY_PAUSED_TRANSACTION, data, reply, 0);
3141        reply.readException();
3142        data.recycle();
3143        reply.recycle();
3144    }
```

又是一个Binder通信过程，所以看AMS的`onTransact`中code=ACTIVITY_PAUSED_TRANSACTION的代码，实际上在AMS父类AMNative中

```java
536        case ACTIVITY_PAUSED_TRANSACTION: {
537            data.enforceInterface(IActivityManager.descriptor);
538            IBinder token = data.readStrongBinder();
539            activityPaused(token);
540            reply.writeNoException();
541            return true;
542        }
```

又调用了AMS的`activityPaused`方法，跟进，擦

```java
6497    @Override
6498    public final void activityPaused(IBinder token) {
6499        final long origId = Binder.clearCallingIdentity();
6500        synchronized(this) {
6501            ActivityStack stack = ActivityRecord.getStackLocked(token);
6502            if (stack != null) {
6503                stack.activityPausedLocked(token, false);
6504            }
6505        }
6506        Binder.restoreCallingIdentity(origId);
6507    }
6508
```

ActivityStack的`activityPausedLocked`

```java
928    final void activityPausedLocked(IBinder token, boolean timeout) {
929        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE,
930            "Activity paused: token=" + token + ", timeout=" + timeout);
931
932        final ActivityRecord r = isInStackLocked(token);
933        if (r != null) {
934            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
935            if (mPausingActivity == r) {

938                completePauseLocked(true);
939            } else {

944                if (r.finishing && r.state == ActivityState.PAUSING) {

947                    finishCurrentActivityLocked(r, FINISH_AFTER_VISIBLE, false);
948                }
949            }
950        }
951    }
```

`completePauseLocked`->`StackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);`

```java
2727    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
2728            Bundle targetOptions) {
2729        if (targetStack == null) {
2730            targetStack = mFocusedStack;
2731        }
2732        // Do targetStack first.
2733        boolean result = false;
2734        if (isFrontStack(targetStack)) {
2735            result = targetStack.resumeTopActivityLocked(target, targetOptions);
2736        }
2737
2738        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
2739            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
2740            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
2741                final ActivityStack stack = stacks.get(stackNdx);
2742                if (stack == targetStack) {
2743                    // Already started above.
2744                    continue;
2745                }
2746                if (isFrontStack(stack)) {
2747                    stack.resumeTopActivityLocked(null);
2748                }
2749            }
2750        }
2751        return result;
2752    }
```

`ActivityStack.resumeTopActivityLocked`->`resumeTopActivityInnerLocked`->`StackSupervisor.startspecificactivitylocked`

```java
1365    void startSpecificActivityLocked(ActivityRecord r,
1366            boolean andResume, boolean checkConfig) {
1367        // Is this activity's application already running?
1368        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
1369                r.info.applicationInfo.uid, true);
1370
1371        r.task.stack.setLaunchTime(r);
1372
1373        if (app != null && app.thread != null) {
1393        }
1394
1395        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
1396                "activity", r.intent.getComponent(), false, false, true);
1397    }
```

AMS.startProcessLocked会启动r.processName=ActivityThread这个进程，则会执行ActivityThread.main方法

## ActivityThread->



```java
5379    public static void main(String[] args) {
5380        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
5381        SamplingProfilerIntegration.start();
5382

5387
5388        Environment.initForCurrentUser();


5398
5399        Process.setArgV0("<pre-initialized>");
5400
5401        Looper.prepareMainLooper();
5402
5403        ActivityThread thread = new ActivityThread();
5404        thread.attach(false);
5405
5406        if (sMainThreadHandler == null) {
5407            sMainThreadHandler = thread.getHandler();
5408        }
5409
5410        if (false) {
5411            Looper.myLooper().setMessageLogging(new
5412                    LogPrinter(Log.DEBUG, "ActivityThread"));
5413        }
5414
5415        // End of event ActivityThreadMain.
5416        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
5417        Looper.loop();
5418
5419        throw new RuntimeException("Main thread loop unexpectedly exited");
5420    }
		



5230    private void attach(boolean system) {
5231        sCurrentActivityThread = this;
5232        mSystemThread = system;
5233        if (!system) {

5243            final IActivityManager mgr = ActivityManagerNative.getDefault();
5244            try {
5245                mgr.attachApplication(mAppThread);
5246            } catch (RemoteException ex) {
5247                // Ignore
5248            }
  			...
			}
  			...
		}
```



`attachApplication`，还是一个套路，AMProxy通过Binder发送，AMS来响应的套路，这次的code码为ATTACH_APPLICATION_TRANSACTION

```JAVA
//AMProxy
3093    public void attachApplication(IApplicationThread app) throws RemoteException
3094    {
3095        Parcel data = Parcel.obtain();
3096        Parcel reply = Parcel.obtain();
3097        data.writeInterfaceToken(IActivityManager.descriptor);
3098        data.writeStrongBinder(app.asBinder());
3099        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
3100        reply.readException();
3101        data.recycle();
3102        reply.recycle();
3103    }

//AMNatvie
502        case ATTACH_APPLICATION_TRANSACTION: {
503            data.enforceInterface(IActivityManager.descriptor);
504            IApplicationThread app = ApplicationThreadNative.asInterface(
505                    data.readStrongBinder());
506            if (app != null) {
507                attachApplication(app);
508            }
509            reply.writeNoException();
510            return true;
511        }
512
  
//AMS
  6245    @Override
6246    public final void attachApplication(IApplicationThread thread) {
6247        synchronized (this) {
6248            int callingPid = Binder.getCallingPid();
6249            final long origId = Binder.clearCallingIdentity();
6250            attachApplicationLocked(thread, callingPid);
6251            Binder.restoreCallingIdentity(origId);
6252        }
6253    }

```

AMS.attachApplicationLocked 又调用了Stack.realStartActivityLocked -> ApplicationThreadProxy.scheduleLaunchActivity->ApplicationThread.scheduleLaunchActivity->ActivityThread.sendMessage->H.handleMessage->handleLaunchActivity->ActivityThread.handleLaunchActivity->performLaunchActivity->mInstrumentation.callActivityOnCreate

















# 参考

[AMS和SystemServer](https://www.cnblogs.com/bastard/p/5770573.html)

[ActivityManager ActivityManagerProxy等关系](http://blog.csdn.net/kc58236582/article/details/50394905)

[ActivityRecord、TaskRecord、ActivityStack](http://blog.csdn.net/kebelzc24/article/details/53747506)

---
title: ViewRootImpl与Activity
date: 2019-06-27 11:27:45
tags: [View]  
categories: Android
---

子线程能不能更新UI？

<!-- more -->   





# 几个涉及的类说明

## Window、PhoneWindow

PhoneWindow是Window子类

setContentView其实就是将View设置到Window上，Activity展示的其实是PhoneWindow上的内容



```java
//Activity.java
final void attach() {
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
}
```

## WindowManager、WindowManagerService

WindowManager 父类是 **ViewManager**    实现类是**WindowManagerImpl**

WindowManager真正实现控制view添加删除更新

WindowManager和WindowManagerService的交互是一个IPC过程

```java
//Window.java 上面的代码块的方法跟进
public void setWindowManager() {
    mAppToken = appToken;
    mAppName = appName;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```



## WindowManagerGlobal

WindowManagerGlobal 真正实现了 WMImpl的 增删View的动作

```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

		@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) 		{
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
}
```



## ViewRootImpl

```java
//WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        root = new ViewRootImpl(view.getContext(), display);
        mRoots.add(root);
      
      	// do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
            }
    }
}
//ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
 			 		// Schedule the first layout -before- adding to the window
          // manager, to make sure we do the relayout before receiving
          // any other events from the system.
          requestLayout();
          //关注这个方法 下面setContentView里边会用到！！！！
          //让decroView的爸爸变成ViewRootImpl
          view.assignParent(this);
          /*
           //View.java
           void assignParent(ViewParent parent) {
              if (mParent == null) {
              mParent = parent;
              }
            }
          */
        }
}
//ViewRootImpl.java
 public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread(); //关注这个方法  检查当前线程用的  即ViewRootImpl来负责线程的检查
            mLayoutRequested = true;
           //关注这个方法  performTraversals就是由他来触发(post-message)  
          //performTraversals 就是触发 measure layout  draw的
          scheduleTraversals(); 
        }
    }
```

## 总结

![](https://github.com/MelonWXD/BlogPicStorage/blob/master/Window关系图.png?raw=true)

来自：<https://www.cnblogs.com/mingfeng002/p/9143831.html>

- 鉴于窗口布局和控件布局的一致性，WindowManager继承并实现了接口ViewManager。
- 使用者可以通过Context.getSystemService(Context.WINDOW_SERVICE)来获取一个WindowManager的实例。这个实例的真实类型是WindowManagerImpl。WindowManagerImpl一旦被创建就确定了通过它所创建的窗口所属哪块屏幕？哪个父窗口？
- WindowManagerImpl除了保存了窗口所属的屏幕以及父窗口以外，没有任何实质性的工作。窗口的管理都交由WindowManagerGlobal的实例完成。
- WindowManagerGlobal在一个进程中只有一个实例。
- WindowManagerGlobal在3个数组中统一管理整个进程中的所有窗口的信息。这些信息包括控件、布局参数以及ViewRootImpl三个元素。
- 除了管理窗口的上述3个元素以外，WindowManagerGlobal将窗口的创建、销毁与布局更新等任务交付给了ViewRootImpl完成。



# handleLaunchActivity

启动Activity流程中  最后会调用到 `handleLaunchActivity`

在内部依次调用`performLaunchActivity` 和 `handleResumeActivity`

```java
//androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  if (activity != null) {
    //这里就是调用到最上面的attach代码块 初始化window windowmanager
		activity.attach(appContext, this, getInstrumentation(), /*...*/);
    //调用生命周期 onCreate          
 		mInstrumentation.callActivityOnCreate(activity, r.state);
 		if (!r.activity.mFinished) {
      	//调用生命周期 onStart
				activity.performStart();
				r.stopped = false;
		}
}
  // Instrumentation.java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        activity.performCreate(icicle);
 }
  //Activity.java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        onCreate(icicle);
        mFragments.dispatchActivityCreated();
}
```



```java
 final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
   //调用生命周期 onResume
   ActivityClientRecord r = performResumeActivity(token, clearHide);
   
		if (r.window == null && !a.mFinished && willBeVisible) {
      r.window = r.activity.getWindow();
      
      if (a.mVisibleFromClient) {
        a.mWindowAdded = true;
        //关注这个方法 就是上面创建ViewRootImpl的方法块  把decorView添加到window中
        //根据上面的分析  add之后就会开始走view measure/layout/draw等实际绘制流程
        wm.addView(decor, l);
      }
    }
 }
public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
 			r.activity.performResume();
}
```

## 总结

上面采用了瞎比倒叙法来说明，可能有点乱，这里总结理一下

`handleResumeActivity` -> `onResume` 

->`WindowManagerImpl.addView(传入的是DecorView)` -> `WindowManagerGlobal.addView()`

->`new ViewRootImpl()` -> `ViewRootImpl.setView()`

->`requestLayout()`



让我们回到最开始的问题：子线程能不能更新UI？

答案是能，在onCreate中更新ui，此时ViewRootImpl 还未被创建(onResume才创建)，所以不会传到ViewRootImpl来实际绘制，更没有`checkThread`来抛出异常，这样在正常流程开始绘制的时候，就会绘制子线程中更新进去的内容

# setContentView

```java
//Activity
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
//PhoneWindow
404    @Override
405    public void setContentView(int layoutResID) {
  //mContentParent 就是 android.R.id.content 的 FrameLayout
409        if (mContentParent == null) {
410            installDecor();
411        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
412            mContentParent.removeAllViews();
413        }
414
415        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
416            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
417                    getContext());
418            transitionTo(newScene);
419        } else {
420            mLayoutInflater.inflate(layoutResID, mContentParent);
421        }
422        mContentParent.requestApplyInsets();//请求绘制
423        final Callback cb = getCallback();
424        if (cb != null && !isDestroyed()) {
425            cb.onContentChanged();
426        }
427        mContentParentExplicitlySet = true;
428    }
//View.java
    /**
     * Ask that a new dispatch of {@link #onApplyWindowInsets(WindowInsets)} be performed.
     */
    public void requestApplyInsets() {
        requestFitSystemWindows();
    }

//这里交给DecorView的mParent   回到上面 ViewRootImpl 里的代码块 
//view.assignParent(this);  可以知道 mParent 就是 ViewRootImpl
	public void requestFitSystemWindows() {
        if (mParent != null) {
            mParent.requestFitSystemWindows();
        }
    }

//ViewRootImpl.java
1215    @Override
1216    public void requestFitSystemWindows() {
1217        checkThread();
1218        mApplyInsetsRequested = true;
1219        scheduleTraversals();
1220    }
```

# scheduleTraversals

不管是`requestLayout` 还是`requestFitSystemWindows` 最后都是来到`scheduleTraversals`方法中

```java
1428    void scheduleTraversals() {
1429        if (!mTraversalScheduled) {
1430            mTraversalScheduled = true;
  							//屏障？
1431            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
1432            mChoreographer.postCallback(
1433                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);

1439        }
1440    }

7181    final class TraversalRunnable implements Runnable {
7182        @Override
7183        public void run() {
7184            doTraversal();
7185        }
7186    }


1451    void doTraversal() {
1452        if (mTraversalScheduled) {
1453            mTraversalScheduled = false;
1454            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

1459
1460            performTraversals();//来啦老弟
  
1466        }
1467    }
```



# performTraversals

是怎么开启每个view的 measure  layout  draw 的?  且听下回分解  饭点到了

> performTraversals()中
>
> 使用performMeasure() 来测量View大小。
>
> 使用performLayout()来确定View的绘制区域。
>
> 使用performDraw()来真正的绘制

# 参考

[android window(二)从getSystemService到WindowManagerGlobal](https://www.cnblogs.com/mingfeng002/p/9143831.html)
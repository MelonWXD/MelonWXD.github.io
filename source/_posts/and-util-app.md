---
title: App相关全局管理类
date: 2017-11-25 20:30:45
tags: [Application]
categories: Android代码库
---
Activity管理类以及App类
<!-- more -->
## 代码
> AppManager.java  

```java

public class AppManager {

    private static Stack<Activity> activityStack;
    private static AppManager instance;

    public Context getAppContext() {
        return appContext;
    }

    private Context appContext;
    private AppManager() {
    }

    /**
     * 单一实例
     */
    public static AppManager getInstance() {
        if (instance == null) {
            instance = new AppManager();
        }
        return instance;
    }

    /**
     * 添加Activity到堆栈
     */
    public void addActivity(Activity activity) {
        if (activityStack == null) {
            activityStack = new Stack<Activity>();
        }
        activityStack.add(activity);
    }

    /**
     * 获取当前Activity（堆栈中最后一个压入的）
     */
    public Activity currentActivity() {
        Activity activity = activityStack.lastElement();
        return activity;
    }

    /**
     * 结束当前Activity（堆栈中最后一个压入的）
     */
    public void finishActivity() {
        Activity activity = activityStack.lastElement();
        finishActivity(activity);
    }

    /**
     * 结束指定的Activity
     */
    public void finishActivity(Activity activity) {
        if (activity != null) {
            activityStack.remove(activity);
            activity.finish();
            activity = null;
        }
    }

    /**
     * 结束指定类名的Activity
     */
    public void finishActivity(Class<?> cls) {
        for (Activity activity : activityStack) {
            if (activity.getClass().equals(cls)) {
                finishActivity(activity);
            }
        }
    }

    /**
     * 结束所有Activity
     */
    public void finishAllActivity() {
        for (int i = 0, size = activityStack.size(); i < size; i++) {
            if (null != activityStack.get(i)) {
                activityStack.get(i).finish();
            }
        }
        activityStack.clear();
    }

    /**
     * 退出应用程序
     */
    public void AppExit(Context context) {
        try {
            finishAllActivity();
            ActivityManager activityMgr = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
            activityMgr.restartPackage(context.getPackageName());
            System.exit(0);
        } catch (Exception e) {
        }
    }

    public void setContext(Context context) {
        this.appContext = context;
    }
}


```

> App 来初始化一些数据库网络等

```
public class App extends Application {

    private static DaoSession mDaoSession;
    private Context mContext;

    @Override
    public void onCreate() {
        super.onCreate();
        mContext = this.getApplicationContext();
        AppManager.getInstance().setContext(mContext);
        setupDatabase(this);
        setupLog();
        setupNet();
        ...



    }





    private void setupDatabase(Context context) {
        DaoMaster.DevOpenHelper openHelper = new DaoMaster.DevOpenHelper(context, DATABASE_WEATHER);
        Database db = openHelper.getWritableDb();
        DaoMaster daoMaster = new DaoMaster(db);
        mDaoSession = daoMaster.newSession();
    }

 

    public static DaoSession getDaoSession() {
        return mDaoSession;
    }



}

```

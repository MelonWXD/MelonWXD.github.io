---
title: Dalvik加载dex文件分析
date: 2018-04-02 21:08:02
tags: [Dalvik]  
categories: 虚拟机
---
分析Dalvik加载dex文件流程
<!-- more -->  

# .dex文件格式

文件格式这个我觉得很死啊，就跟之前[分析ELF文件格式](https://melonwxd.github.io/2017/11/19/inject-1-elf/)一样，按照每个结构体对应的值来找，很无脑但是也很容易出错，我先贴个非虫的图吧，后续有机会再补充。

非虫大佬的神图:

![](http://img.blog.csdn.net/20160215152236880?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



# 从app_process到AndroidRuntime

一切又要从zygote进程启动开始说起，结合之前的[分析](https://melonwxd.github.io/2017/12/08/Android启动流程/ )，在init.rc中解析zygote的时候，zygote64.rc的内容如下：

```cpp
1 service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
2    class main    //声明service所属class的名字
3    socket zygote stream 660 root system
4    onrestart write /sys/android_power/request_state wake
5    onrestart write /sys/power/state on
6    onrestart restart media
7    onrestart restart netd
8    writepid /dev/cpuset/foreground/tasks
```



可以知道zygote要执行的程序是`system/bin/app_process64`，他的源码位置在[app_main.cpp](http://androidxref.com/4.4.4_r1/xref/frameworks/base/cmds/app_process/app_main.cpp)，截取部分跟dvm相关的代码：

```cpp
138int main(int argc, char* const argv[])
139{
  
177    AppRuntime runtime;

187    int i = runtime.addVmArguments(argc, argv);

222    if (zygote) {
223        runtime.start("com.android.internal.os.ZygoteInit",
224                startSystemServer ? "start-system-server" : "");
238 }
```

当执行app_main.cpp的`main`函数为zygote进程的时候就会调用AppRuntime的start方法。

AppRuntime这个类如下：

```cpp
32class AppRuntime : public AndroidRuntime
33{

56
57    virtual void onVmCreated(JNIEnv* env)
58    {

84    }
85
86    virtual void onStarted()
87    {

96    }
97
98    virtual void onZygoteInit()
99    {

106    }
107
108    virtual void onExit(int code)
109    {

116    }

124};
```

继承自[AndroidRuntime](http://androidxref.com/4.4.4_r1/xref/frameworks/base/core/jni/AndroidRuntime.cpp)，跟进父类的start方法瞅瞅

```cpp
797/*
798 * Start the Android runtime.  This involves starting the virtual machine
799 * and calling the "static void main(String[] args)" method in the class
800 * named by "className".
      ↑↑↑↑↑注释自己看 说的很简洁↑↑↑↑↑
804 */
805void AndroidRuntime::start(const char* className, const char* options)
806{

834    /* start the virtual machine */
835    JniInvocation jni_invocation;
836    jni_invocation.Init(NULL);
  
837    JNIEnv* env;
838    if (startVm(&mJavaVM, &env) != 0) {
839        return;
840    }
841    onVmCreated(env);
842
875    char* slashClassName = toSlashClassName(className);
876    jclass startClass = env->FindClass(slashClassName);
877    if (startClass == NULL) {
878        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
879        /* keep going */
880    } else {
881        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
882            "([Ljava/lang/String;)V");
			...
	   }
902}
```

## JniInvocation:加载libdvm.so

[JniInvocation.cpp](http://androidxref.com/4.4.4_r1/xref/libnativehelper/JniInvocation.cpp)中的Init方法

```cpp
4 static const char* kLibraryFallback = "libdvm.so";

56 bool JniInvocation::Init(const char* library) {
57 #ifdef HAVE_ANDROID_OS
58  char default_library[PROPERTY_VALUE_MAX];
59  property_get(kLibrarySystemProperty, default_library, kLibraryFallback);
60 #else
61  const char* default_library = kLibraryFallback;
62 #endif
63  if (library == NULL) {
64    library = default_library;
65  }
  
67  handle_ = dlopen(library, RTLD_NOW);
  
88  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetDefaultJavaVMInitArgs_),
89                  "JNI_GetDefaultJavaVMInitArgs")) {
90    return false;
91  }
92  if (!FindSymbol(reinterpret_cast<void**>(&JNI_CreateJavaVM_),
93                  "JNI_CreateJavaVM")) {
94    return false;
95  }
96  if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetCreatedJavaVMs_),
97                  "JNI_GetCreatedJavaVMs")) {
98    return false;
99  }
100  return true;
101}
```

粗略看下即可，针对5.0之前的系统，通过line67的`dlopen`来加载`"libdvm.so"`，同时还查找了三个函数，记着阿，下面要用。

##startVm:启动虚拟机

```cpp
435int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv)
436{
437    int result = -1;
438    JavaVMInitArgs initArgs;
439    JavaVMOption opt;
      //属性定义与配置一大堆

767    /*
768     * Initialize the VM.
769     *
770     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
771     * If this call succeeds, the VM is ready, and we can start issuing
772     * JNI calls.
773     */
774    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
775        ALOGE("JNI_CreateJavaVM failed\n");
776        goto bail;
777    }
778
779    result = 0;
780
781 bail:
782    free(stackTraceFile);
783    return result;
784 }

```

最核心的方法：[JNI_CreateJavaVM](http://androidxref.com/4.4.4_r1/xref/dalvik/vm/Jni.cpp)，肯定是要略过不讲的，等以后另开一篇，反正这里虚拟机就启动了。

## JniEnv加载

在`startVm`方法返回之后，就拿到了虚拟机的JNI接口`JNIEnv* env;`，就可以通过下面的FindClass和GetStaticMethodID来加载参数classname 值为`com.android.internal.os.ZygoteInit`以及其main函数。

```java
876    jclass startClass = env->FindClass(slashClassName);
  
881        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
882            "([Ljava/lang/String;)V");
```

 

# 类的加载

**todo: 上面的Jni调用到类的加载，这个过程我暂时还没梳理清楚**

但这并不妨碍我分析下面2个函数 -m-

Jvm对new操作符对应的OP_NEW_INSTANCE处理如下，地址在[/dalvik/vm/mterp/out/InterpC-portable.cpp](http://androidxref.com/4.4.4_r1/xref/dalvik/vm/mterp/out/InterpC-portable.cpp)

```cpp
1649/* File: c/OP_NEW_INSTANCE.cpp */
1650HANDLE_OPCODE(OP_NEW_INSTANCE /*vAA, class@BBBB*/)
1651    {
1652        ClassObject* clazz;
1653        Object* newObj;
1654
1655        EXPORT_PC();
1656
1657        vdst = INST_AA(inst);
1658        ref = FETCH(1);
1659        ILOGV("|new-instance v%d,class@0x%04x", vdst, ref);
1660        clazz = dvmDexGetResolvedClass(methodClassDex, ref);
1661        if (clazz == NULL) {
1662            clazz = dvmResolveClass(curMethod->clazz, ref, false);
1665        }
1667        if (!dvmIsClassInitialized(clazz) && !dvmInitClass(clazz))
1668            GOTO_exceptionThrown();
1691        newObj = dvmAllocObject(clazz, ALLOC_DONT_TRACK);

1694        SET_REGISTER(vdst, (u4) newObj);
1695    }
1696    FINISH(2);
1697OP_END
```



## dvmResolveClass

定义在[/dalvik/vm/oo/Resolve.cpp](http://androidxref.com/4.4.4_r1/xref/dalvik/vm/oo/Resolve.cpp#63)

```cpp
63ClassObject* dvmResolveClass(const ClassObject* referrer, u4 classIdx,
64    bool fromUnverifiedConstant)
65{
66    DvmDex* pDvmDex = referrer->pDvmDex;
67    ClassObject* resClass;
68    const char* className;
69
// 先查看是否要加载的类已经被解析过了
74    resClass = dvmDexGetResolvedClass(pDvmDex, classIdx);
75    if (resClass != NULL)
76        return resClass;
//没有的话就 就解析dex文件 
90    className = dexStringByTypeIdx(pDvmDex->pDexFile, classIdx);
  //判断是不是原始类型  primitive type
91    if (className[0] != '\0' && className[1] == '\0') {
92        /* primitive type */   
93        resClass = dvmFindPrimitiveClass(className[0]);
94    } else {
  // 不是原始类型就调用dvmFindClassNoInit
95        resClass = dvmFindClassNoInit(className, referrer->classLoader);
96    }
97
98    if (resClass != NULL) {
  //这里就是我在热更新总结的文章里说道的，dvm的检查代码
  //https://melonwxd.github.io/2018/03/16/%E7%83%AD%E6%9B%B4%E6%96%B0%E5%85%A8%E8%A7%A3/
141        }
		//解析完该类  缓存起来 下次就直接通过line74的函数返回了
154        dvmDexSetResolvedClass(pDvmDex, classIdx, resClass);
155    } 

162    return resClass;
163}
```

总体流程已经注释好了，主要跟进[dvmFindClassNoInit](http://androidxref.com/4.4.4_r1/xref/dalvik/vm/oo/Class.cpp#1285)函数中进一步了解流程

```cpp
1285ClassObject* dvmFindClassNoInit(const char* descriptor,
1286        Object* loader)
1287{

1293    if (*descriptor == '[') {
1294        /*
1295         * Array class.  Find in table, generate if not found.
1296         */
1297        return dvmFindArrayClass(descriptor, loader);
1298    } else {
1299        /*
1300         * Regular class.  Find in table, load if not found.
1301         */
1302        if (loader != NULL) {
  				//通常的类都是走这个函数，应该都有classloader
1303            return findClassFromLoaderNoInit(descriptor, loader);
1304        } else {
  				//根据类名来看应该是系统类走这个方法 如 java.lang包下的？ 没有深究
1305            return dvmFindSystemClassNoInit(descriptor);
1306        }
1307    }
1308}
```

继续跟进函数 findClassFromLoaderNoInit

```cpp
1316 static ClassObject* findClassFromLoaderNoInit(const char* descriptor,
1317    Object* loader)
1318 {

1322    Thread* self = dvmThreadSelf();
1323

1344    char* dotName = NULL;
1345    StringObject* nameObj = NULL;
1346
1347    /* 转换符号 convert "Landroid/debug/Stuff;" to "android.debug.Stuff" */
1348    dotName = dvmDescriptorToDot(descriptor);

1353    nameObj = dvmCreateStringFromCstr(dotName);

1359    dvmMethodTraceClassPrepBegin();
1360
1361    /*
1362     * Invoke loadClass().  This will probably result in a couple of
1363     * exceptions being thrown, because the ClassLoader.loadClass()
1364     * implementation eventually calls VMClassLoader.loadClass to see if
1365     * the bootstrap class loader can find it before doing its own load.
1366     */

1368    {
1369        const Method* loadClass =
1370            loader->clazz->vtable[gDvm.voffJavaLangClassLoader_loadClass];
1371        JValue result;
1372        dvmCallMethod(self, loadClass, loader, &result, nameObj);
1373        clazz = (ClassObject*) result.l;
1374
1375        dvmMethodTraceClassPrepEnd();
1376        Object* excep = dvmGetException(self);
1377        if (excep != NULL) {
1382            dvmAddTrackedAlloc(excep, self);
1383            dvmClearException(self);
1384            dvmThrowChainedNoClassDefFoundError(descriptor, excep);
1385            dvmReleaseTrackedAlloc(excep, self);
1386            clazz = NULL;
1387            goto bail;
1388        } else if (clazz == NULL) {
1389            ALOGW("ClassLoader returned NULL w/o exception pending");
1390            dvmThrowNullPointerException("ClassLoader returned null");
1391            goto bail;
1392        }
1393    }
1394
1395    /* not adding clazz to tracked-alloc list, because it's a ClassObject */
1396
1397    dvmAddInitiatingLoader(clazz, loader);

1402 bail:
1403    dvmReleaseTrackedAlloc((Object*)nameObj, NULL);
1404    free(dotName);
1405    return clazz;
1406 }
1407
```



## dvmInitClass

# 参考

[深入理解Android（二）：Java虚拟机Dalvik](http://www.infoq.com/cn/articles/android-in-depth-dalvik)

[老罗的分析](https://blog.csdn.net/luoshengyang/article/details/41688319)
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

最核心的方法：[JNI_CreateJavaVM](http://androidxref.com/4.4.4_r1/xref/dalvik/vm/Jni.cpp)，肯定是要略过不讲的，等以后另开一篇。

# 类的加载

dvmResolveClass

dvmInitClass

# 参考

http://www.infoq.com/cn/articles/android-in-depth-dalvik
---
title: ClassLoader 相关面试题
date: 2017-10-17 17:43:11 
tags: [ClassLoader]
categories: 虚拟机
---

检测你是否掌握了[ClassLoader的基础知识](https://melonwxd.github.io/2017/10/17/classloader1/)
<!-- more -->

## 能不能自己写个类叫java.lang.System？  
有这么一道面试题：能不能自己写个类叫java.lang.System？  
先假设，如果我们要写一个java.lang.System这个类，结合上述知识，能否拿出什么合理的方案呢？  
### 分析1：双亲委托原则
想要 jvm 载入 java.lang.System 这个类，根据双亲委托原则，会先去内存查找，看是否已经加载了。真正的java.lang.System是由BootstrapClassloader加载的（由System.class.getClassLoader()==null 可以判断出），等你去内存找的时候，就直接返回正确的java.lang.System了。
那么是不是说 没有办法自己写一个叫做java.lang.System的类了呢？
不，只能说明，遵循双亲委托原则，是没办法实现的，那么不遵循不就行了呗。
### 分析2：重写findClass
双亲委托原则根据之前的分析，也是基于java代码在findClass方法中实现的，如果我们重写findClass里面的方法，我findClass的时候不去内存找，我直接调用findClass去读取文件，自行构建class，加载到内存行不行？  
说干就干，写完感觉还不错，貌似可以的样子
```java
    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        if(name.equals("java.lang.System"))
            return findClass(name);
        return super.loadClass(name);
    }
```
我们自己写的System.java
```java
package java.lang;

public class System {
    public static void gc() {
         
    }
}

```
自定义的ClassLoader的main方法(完整代码见 [ClassLoader的分析与使用](https://melonwxd.github.io/2017/10/17/classloader1/))
```java
    public static void main(String[] args) {
        ClassLoaderTest classLoaderTest = new ClassLoaderTest("/home/duoyi/Desktop/ClassLoaderDemo");
        try {
            Class c = classLoaderTest.findClass("java.lang.System");
            Object obj = c.newInstance();
            Method method1 = c.getDeclaredMethod("gc",null);
            method1.invoke(obj, null);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
```
运行一下看看，输出：
```
java.lang.SecurityException: Prohibited package name: java.lang
	at java.lang.ClassLoader.preDefineClass(ClassLoader.java:662)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:761)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
	at PckA.ClassLoaderTest.findClass(ClassLoaderTest.java:49)
	at PckA.ClassLoaderTest.main(ClassLoaderTest.java:69)
```
抛出一个安全异常：包名为禁用包名  
进入defineClass中的preDinfeClass方法看一看
```java
private ProtectionDomain preDefineClass(String name, ProtectionDomain pd)
    {
        // Note:  Checking logic in java.lang.invoke.MemberName.checkForTypeAlias
        // relies on the fact that spoofing is impossible if a class has a name
        // of the form "java.*"
        if ((name != null) && name.startsWith("java.")) {
            throw new SecurityException
                ("Prohibited package name: " +
                 name.substring(0, name.lastIndexOf('.')));
        }
    }
```
这就没办法了，他在内部做了限制，不允许java开头的包名被defineClass方法构造，而且defineClass是final方法，也无法通过重写来绕过，所以最终答案就是，不能

---
title: Gradle构建学习
date: 2019-06-05 10:40:22
tags: [Gradle,Android构建]  
categories: Android
---
Gradle入门

<!-- more -->  

# 综述
## LifeCycle

### Initialization

Gradle supports single and multi-project builds. During the initialization phase, Gradle determines which projects are going to take part in the build, and creates a [Project](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html) instance for each of these projects.

确定参与构建的项目，为每个项目创建一个`project`实例。Android中就是根据`setting.grade`中声明的项目来创建。具体步骤见下面CoreType-Project。

### Configuration

During this phase the project objects are configured. The build scripts of *all* projects which are part of the build are executed.

执行每个项目自己的build scripts构建脚本。Android中就是执行个module中的`build.grade`。

### Execution

Gradle determines the subset of the tasks, created and configured during the configuration phase, to be executed. The subset is determined by the task name arguments passed to the `gradle` command and the current directory. Gradle then executes each of the selected tasks.

决定要执行的任务的子集(第二阶段中创建和配置的)。

## CoreType

### Project

Gradle对外的接口，通过这个接口来编程使用Gradle的各种功能。

Project和`build.gradle`文件是一对一关系，在`Initialization`时期，Gradle会为每个参与构建的项目创建一个`project`实例。步骤如下：

1. 为本次构造创建一个`Settings`实例

2. 读取`settings.gradle`来配置上面的`Settings`对象
3. 通过上面的`Settings`对象来创建`Project`实例
4. 执行每个项目的`build.gradle`来配置每个`Project`实例

### Task

`Project`可以算是由一堆`Task`构成的集合。每个`Task`各司其职，比如编译代码、跑单元测试、压缩对齐等。

[`TaskContainer.create(java.lang.String)`](https://docs.gradle.org/4.4/javadoc/org/gradle/api/tasks/TaskContainer.html#create(java.lang.String))是用来添加`Task`的，通过[`TaskCollection.getByName(java.lang.String)`](https://docs.gradle.org/4.4/javadoc/org/gradle/api/tasks/TaskCollection.html#getByName(java.lang.String))也可以获取到某个具体的`Task`实例。



### Plugins

`Plugins`可以用于模块化与项目配置复用。

Plugins can be applied using the [`PluginAware.apply(java.util.Map)`](https://docs.gradle.org/4.4/dsl/org.gradle.api.plugins.PluginAware.html#org.gradle.api.plugins.PluginAware:apply(java.util.Map)) method, or by using the [`PluginDependenciesSpec`](https://docs.gradle.org/4.4/dsl/org.gradle.plugin.use.PluginDependenciesSpec.html) plugins script block.



# Android 编译流程

图例可以看[之前的博客](https://melonwxd.github.io/2017/10/10/dalvik-art/)

这里主要基于Gradle的命令展开

用Gradle打个debug包的命令如下，—info是为了打印出详细信息  并输出到文件 方便观察

> ./gradlew --info assembleDebug  > ~/Desktop/gradle.info

开始一步步看日志吧

## Initialization

读取settings

```java
Starting Build
Settings evaluated using settings file '/Users/wengxiaodong/AndroidStudioProjects/VideoLearn/settings.gradle'.
Projects loaded. Root project using build file '/Users/wengxiaodong/AndroidStudioProjects/VideoLearn/build.gradle'.
Included projects: [root project 'VideoLearn', project ':app']
```

## Configuration

```java
> Configure project :
Evaluating root project 'VideoLearn' using build file '/Users/wengxiaodong/AndroidStudioProjects/VideoLearn/build.gradle'.

> Configure project :app
Evaluating project ':app' using build file '/Users/wengxiaodong/AndroidStudioProjects/VideoLearn/app/build.gradle'.
```



```java
All projects evaluated.
Selected primary task 'assembleDebug' from project :
Tasks to be executed: [
  task ':app:preBuild', 
  task ':app:preDebugBuild', 
  task ':app:compileDebugAidl',     //编译AIDL
  task ':app:compileDebugRenderscript', 
  task ':app:checkDebugManifest',  
  task ':app:generateDebugBuildConfig',  //BuildConfig.java
  task ':app:prepareLintJar', 
  task ':app:generateDebugSources', 
  task ':app:javaPreCompileDebug', 
  task ':app:mainApkListPersistenceDebug', 
  task ':app:generateDebugResValues',  //Res
  task ':app:generateDebugResources',   
  task ':app:mergeDebugResources', 
  task ':app:createDebugCompatibleScreenManifests', 
  task ':app:processDebugManifest',  //Manifest
  task ':app:processDebugResources', 
  task ':app:compileDebugJavaWithJavac',  //javac编译
  task ':app:compileDebugSources', 
  task ':app:mergeDebugShaders', 
  task ':app:compileDebugShaders',   
  task ':app:generateDebugAssets',   //生成assets
  task ':app:mergeDebugAssets',      //合并assets
  task ':app:checkDebugDuplicateClasses', 
  task ':app:mergeExtDexDebug',   //dex
  task ':app:mergeLibDexDebug', 
  task ':app:transformClassesWithDexBuilderForDebug', 
  task ':app:mergeProjectDexDebug', 
  task ':app:validateSigningDebug', 
  task ':app:signingConfigWriterDebug', 
  task ':app:mergeDebugJniLibFolders',  //jni
  task ':app:transformNativeLibsWithMergeJniLibsForDebug', 
  task ':app:transformNativeLibsWithStripDebugSymbolForDebug', 
  task ':app:processDebugJavaRes', 
  task ':app:transformResourcesWithMergeJavaResForDebug', //resources.arsc
  task ':app:packageDebug',  //打包
  task ':app:assembleDebug'  //apk
] 
```



## Execution

log太多自行查看  就是执行上面的task

# 参考

[写给 Android 开发者的 Gradle 系列（一）基本姿势](https://blog.csdn.net/ziwang_/article/details/80276839)

[通过gradle生成apk的过程](https://blog.csdn.net/kylewo/article/details/82632154)

[Gradle Build Language Reference](https://docs.gradle.org/4.4/dsl/)

[Android Plugin DSL Reference](https://google.github.io/android-gradle-dsl/3.1/)
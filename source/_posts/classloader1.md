---
title: ClassLoader的分析与使用
date: 2017-10-17 11:53:31  
tags: [ClassLoader,HotFix]  
categories: JVM  
---
深入学习ClassLoader原理与学习自定义ClassLoader的使用
<!-- more -->

# JAVA自带的三个类加载器
Java语言系统自带有三个类加载器: 
- Bootstrap ClassLoader 最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变Bootstrap ClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。 
- Extention ClassLoader 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。 
- Appclass Loader也称为SystemAppClass 加载当前应用的classpath的所有类。

这三个类加载器各自对应加载的jar包和class文件的位置
```java
    public static void main(String[] args) {
        System.out.println("BootstrapClassLoader加载Jar包路径: "+System.getProperty("sun.boot.class.path"));
        System.out.println("ExtClassLoader加载Jar包路径: "+System.getProperty("java.ext.dirs"));
        System.out.println("AppClassLoader加载Jar包路径: "+System.getProperty("java.class.path"));
    }
```
输出
```
BootstrapClassLoader加载Jar包路径: /opt/jdk1.8.0_144/jre/lib/resources.jar:/opt/jdk1.8.0_144/jre/lib/rt.jar:/opt/jdk1.8.0_144/jre/lib/sunrsasign.jar:/opt/jdk1.8.0_144/jre/lib/jsse.jar:/opt/jdk1.8.0_144/jre/lib/jce.jar:/opt/jdk1.8.0_144/jre/lib/charsets.jar:/opt/jdk1.8.0_144/jre/lib/jfr.jar:/opt/jdk1.8.0_144/jre/classes
ExtClassLoader加载Jar包路径: /opt/jdk1.8.0_144/jre/lib/ext:/usr/java/packages/lib/ext
AppClassLoader加载Jar包路径: /opt/jdk1.8.0_144/jre/lib/charsets.jar:/opt/jdk1.8.0_144/jre/lib/deploy.jar:/opt/jdk1.8.0_144/jre/lib/ext/cldrdata.jar:/opt/jdk1.8.0_144/jre/lib/ext/dnsns.jar:/opt/jdk1.8.0_144/jre/lib/ext/jaccess.jar:/opt/jdk1.8.0_144/jre/lib/ext/jfxrt.jar:/opt/jdk1.8.0_144/jre/lib/ext/localedata.jar:/opt/jdk1.8.0_144/jre/lib/ext/nashorn.jar:/opt/jdk1.8.0_144/jre/lib/ext/sunec.jar:/opt/jdk1.8.0_144/jre/lib/ext/sunjce_provider.jar:/opt/jdk1.8.0_144/jre/lib/ext/sunpkcs11.jar:/opt/jdk1.8.0_144/jre/lib/ext/zipfs.jar:/opt/jdk1.8.0_144/jre/lib/javaws.jar:/opt/jdk1.8.0_144/jre/lib/jce.jar:/opt/jdk1.8.0_144/jre/lib/jfr.jar:/opt/jdk1.8.0_144/jre/lib/jfxswt.jar:/opt/jdk1.8.0_144/jre/lib/jsse.jar:/opt/jdk1.8.0_144/jre/lib/management-agent.jar:/opt/jdk1.8.0_144/jre/lib/plugin.jar:/opt/jdk1.8.0_144/jre/lib/resources.jar:/opt/jdk1.8.0_144/jre/lib/rt.jar:/home/duoyi/IdeaProjects/ClassLoaderTest/out/production/ClassLoaderTest:/home/duoyi/idea-IC-172.4155.36/lib/idea_rt.jar
```



 ## 父加载器和父类
 查阅ClassLoader源码中构造方法
 ```java
private final ClassLoader parent;
private ClassLoader(Void unused, ClassLoader parent) {
    this.parent = parent;
    ...
}
protected ClassLoader() {
    this(checkCreateClassLoader(), getSystemClassLoader());
}
public final ClassLoader getParent() {
    if (parent == null)
        return null;
    return parent;
}
 ```
从构造方法可以知道每个类加载器都有一个parent变量来代表父加载器，所以父加载器并不是继承关系上的父类。当调用的是无参的构造方法时，会由系统默认创建一个ClassLoader来作为当前类加载器的parent，实际上默认就是AppClassLoader。  
把各ClassLoader的父加载器打印出来看看：
```java
ClassLoader cl = ClassLoaderTest.class.getClassLoader();
System.out.println(cl.toString());
System.out.println(cl.getParent().toString());
System.out.println(cl.getParent().getParent().toString());
```
输出：
```
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@677327b6
Exception in thread "main" java.lang.NullPointerException
	at PckA.ClassLoaderTest.main(ClassLoaderTest.java:86)
```
可以看到一般我们继承ClassLoader来实现的自定义ClassLoader的父加载器，都是AppClassLoader，AppClassLoader的父加载器是ExtCLassLoader，但是ExtClassLoader居然不存在父加载器，看构造方法就知道每个ClassLoader都是有父加载器的，这不是互相矛盾了。其实不然，ExtClassLoader的父加载器就是BootstrapClassLoader，但是Bootstrap是通过C++实现的，所以Java无法拿到它的引用，自然为null了。


 继承关系图：  

 ![](http://img.blog.csdn.net/20170211112754197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJpYmx1ZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 全盘负责与双亲委托

全盘负责 是指当一个ClassLoader装载一个类时，除非显示地使用另一个ClassLoader，则该类所依赖及引用的类也由这个CladdLoader载入。
真正加载class字节码文件生成Class对象由“双亲委派”机制完成。
“双亲委派”机制加载Class的具体过程是：
1. 源ClassLoader先判断该Class是否已加载，如果已加载，则返回Class对象；如果没有则委托给父类加载器。
2. 父类加载器判断是否加载过该Class，如果已加载，则返回Class对象；如果没有则委托给祖父类加载器。
3. 依此类推，直到始祖类加载器（BootstrapClassLoader）。
4. 始祖类加载器判断是否加载过该Class，如果已加载，则返回Class对象；如果没有则尝试从其对应的类路径下寻找class字节码文件并载入。如果载入成功，则返回Class对象；如果载入失败，则委托给始祖类加载器的子类加载器。
5. 始祖类加载器的子类加载器尝试从其对应的类路径下寻找class字节码文件并载入。如果载入成功，则返回Class对象；如果载入失败，则委托给始祖类加载器的孙类加载器。
6. 依此类推，直到源ClassLoader。  
  源ClassLoader尝试从其对应的类路径下寻找class字节码文件并载入。如果载入成功，则返回Class对象；如果载入失败，源ClassLoader不会再委托其子类加载器，而是抛出异常。 
  “双亲委派”机制只是Java推荐的机制，并不是强制的机制。  
  我们可以继承java.lang.ClassLoader类，实现自己的类加载器。如果想保持双亲委派模型，就应该重写findClass(name)方法；如果想破坏双亲委派模型，可以重写loadClass(name)方法。




# 自定义ClassLoader

## findClass、defineClass和loadClass
通常自定义ClassLoader，我们都要重写findClass方法，在其中调用defineClass来返回我们想要加载的特定的那个类
```java
    /**
     * Finds the class with the specified <a href="#name">binary name</a>.
     * This method should be overridden by class loader implementations that
     * follow the delegation model for loading classes, and will be invoked by
     * the {@link #loadClass <tt>loadClass</tt>} method after checking the
     * parent class loader for the requested class.  The default implementation
     * throws a <tt>ClassNotFoundException</tt>.
     */
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
```
defineClass就不深究源码了，根据参数就知道，根据类名、bytes[]来重新构造一个Class类，这个bytes就是findClass中找到class文件后，使用流读取进来写入到byte[]中。
```java
	/*
     *
     * @param  name
     *         The expected <a href="#name">binary name</a> of the class, or
     *         <tt>null</tt> if not known
     *         
     * @param  b
     *         The bytes that make up the class data. The bytes in positions
     *         <tt>off</tt> through <tt>off+len-1</tt> should have the format
     *         of a valid class file as defined by
     *         <cite>The Java&trade; Virtual Machine Specification</cite>.
     *
     * @param  off
     *         The start offset in <tt>b</tt> of the class data
     *
     * @param  len
     *         The length of the class data
     *
     * @param  protectionDomain
     *         The ProtectionDomain of the class
     *         
     */
    protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                         ProtectionDomain protectionDomain)
        throws ClassFormatError
    {
        protectionDomain = preDefineClass(name, protectionDomain);
        String source = defineClassSourceLocation(protectionDomain);
        Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
        postDefineClass(c, protectionDomain);
        return c;
    }
```
loadClass 则体现了上述的双亲委托机制，一般来说是无需改动，为什么说一般，因为后面要改..
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            Class<?> c = findLoadedClass(name);//源ClassLoader先判断该Class是否已加载
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);//父类加载器判断是否加载过该Class
                    } else {
                        c = findBootstrapClassOrNull(name);//父类为null时即为BootstrapClassLoader
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);//向上查找，向下加载又回到源ClassLoader的findClass方法中

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
## 实例
编写要加载的类 Test.java，其中有静态方法main和实例方法fun，尤其要注意这包名：A.B.C
```java
package A.B.C;
public class Test {
    public static void main(String[] args){
        System.out.println("this is main method from Test");
    }
    public void fun(){
        System.out.println("this is fun method from Test");
    }
}
```
然后将该文件放到 /home/duoyi/Desktop/ClassLoaderDemo/A/B/C 下，在终端里通过javac编译一下得到class文件

编写自定义的ClassLoader类，代码写的很清楚了，也是按照上述流程来：
1. 继承ClassLoader
2. 重写findClass，并在内部通过defineClass创建Class实例
3. 通过反射调用方法  


```java

public class ClassLoaderTest extends ClassLoader {

    private String mLibPath;

    public ClassLoaderTest(String path) { 
        mLibPath = path;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException { 

        String fileName = getFileName(name);

        File file = new File(mLibPath, fileName);

        try {
            FileInputStream is = new FileInputStream(file);

            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            int len = 0;
            try {
                while ((len = is.read()) != -1) {
                    bos.write(len);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }

            byte[] data = bos.toByteArray();
            is.close();
            bos.close();

            return defineClass(name, data, 0, data.length);

        } catch (IOException e) { 
            e.printStackTrace();
        }

        return super.findClass(name);
    }

    //将包名转换为实际路径
    private String getFileName(String name) {
        name = name.replaceAll("\\.","/");
        return name+".class";
    }


    public static void main(String[] args) {
        ClassLoaderTest classLoaderTest = new ClassLoaderTest("/home/duoyi/Desktop/ClassLoaderDemo");
        try {

            Class c = classLoaderTest.findClass("A.B.C.Test");
            Object obj = c.newInstance();
            Method method1 = c.getDeclaredMethod("main",String[].class);
            Method method2 = c.getDeclaredMethod("fun",null);
            method1.invoke(obj, (Object) new String[]{});
            method2.invoke(obj,null);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```
运行 输出：
```
this is main method from Test
this is fun method from Test
```

### 考一考  
不小心篇幅写的太多了，详情请参考[这里](https://melonwxd.github.io/2017/10/17/classloader2/)，检验你对上述知识的了解程度。

# Android的ClassLoader

## BaseDexClassLoader

[源码](http://androidxref.com/6.0.1_r10/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)

```java

32    /**
33     * Constructs an instance.
34     *
35     * @param dexPath the list of jar/apk files containing classes and
36     * resources, delimited by {@code File.pathSeparator}, which
37     * defaults to {@code ":"} on Android
38     * @param optimizedDirectory directory where optimized dex files
39     * should be written; may be {@code null}
40     * @param libraryPath the list of directories containing native
41     * libraries, delimited by {@code File.pathSeparator}; may be
42     * {@code null}
43     * @param parent the parent class loader
44     */
45    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
46            String libraryPath, ClassLoader parent) {
47        super(parent);
48        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
49    }

```

    /*第一个参数是jar包或apk的路径，
    第二个参数是将这些jar包或apk优化后成为dex的存储路径，
    第三个参数是本地库的路径，
    第四个参数是父装载器，父装载器有什么用呢？当我们loadClass的时候，会优先用父装载器去loadclass*/
- dexPath：目标类所在的Apk或者Jar文件路径
- optimizedDirectory：我在[这篇文章](https://melonwxd.github.io/2017/10/10/dalvik-art/)里贴过apk安装流程图，apk中的dex会被解压出来，并优化成ODex文件，这个参数就是解压路径。ClassLoader只能加载内部存储路径中的dex文件，所以这个路径必须为内部路径
- libraryPath：本地库的路径，
- parent：父加载器


BaseDexClassLoader有2个子类，DexClassLoader和PathClassLoader，这两个都只重写了BaseDexClassLoader的构造而已，具体的加载逻辑还是在BaseDexClassLoader中。

### FindClass

跟Java一样的加载流程，我们具体看看BaseDex的加载逻辑

```java
30  private final DexPathList pathList;
...
51    @Override
52    protected Class<?> findClass(String name) throws ClassNotFoundException {
53        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
54        Class c = pathList.findClass(name, suppressedExceptions);//在这里通过pathList查找
55        if (c == null) {
56            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
57            for (Throwable t : suppressedExceptions) {
58                cnfe.addSuppressed(t);
59            }
60            throw cnfe;
61        }
62        return c;
63    }
```

 跟进查看DexPathList的`findClass`

```java
61    private final Element[] dexElements;
...
321    /**
322     * Finds the named class in one of the dex files pointed at by
323     * this instance. This will find the one in the earliest listed
324     * path element. If the class is found but has not yet been
325     * defined, then this method will define it in the defining
326     * context that this instance was constructed with.
327     *
328     * @param name of class to find
329     * @param suppressed exceptions encountered whilst finding the class
330     * @return the named class or {@code null} if the class is not
331     * found in any of the dex files
332     */
333    public Class findClass(String name, List<Throwable> suppressed) {
334        for (Element element : dexElements) {
335            DexFile dex = element.dexFile;
336
337            if (dex != null) {
338                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
339                if (clazz != null) {
340                    return clazz;
341                }
342            }
343        }
344        if (dexElementsSuppressedExceptions != null) {
345            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
346        }
347        return null;
348    }
```

遍历一个Element数组，Element持有DexFile实例，通过调用每个DexFile实例的`loadClassBinaryName`来根据`name`载入，实际上内部是调用了`defineClass`方法，来创建类，如果载入成功则返回，否则返回null。

**这里基本可以找到热更的思路了，通过反射把我们的增量class添加到这个Element[]，并保证比要更新的同名类更早加载即可**

Qzone团队的图：

![](http://img.blog.csdn.net/20160315141956321)

### PathClassLoader

Android系统是通过PathClassLoader加载系统类和已安装的应用的。 

```java
26    /**
27     * Creates a {@code PathClassLoader} that operates on a given list of files
28     * and directories. This method is equivalent to calling
29     * {@link #PathClassLoader(String, String, ClassLoader)} with a
30     * {@code null} value for the second argument (see description there).
31     *
32     * @param dexPath the list of jar/apk files containing classes and
33     * resources, delimited by {@code File.pathSeparator}, which
34     * defaults to {@code ":"} on Android
35     * @param parent the parent class loader
36     */
 
37    public PathClassLoader(String dexPath, ClassLoader parent) {
38        super(dexPath, null, null, parent);
39    }

```



### DexClassLoader

DexClassPath则可以从一个jar包或者未安装的apk中加载dex

```java
37    /**
38     * Creates a {@code DexClassLoader} that finds interpreted and native
39     * code.  Interpreted classes are found in a set of DEX files contained
40     * in Jar or APK files.
41     *
42     * <p>The path lists are separated using the character specified by the
43     * {@code path.separator} system property, which defaults to {@code :}.
44     *
45     * @param dexPath the list of jar/apk files containing classes and
46     *     resources, delimited by {@code File.pathSeparator}, which
47     *     defaults to {@code ":"} on Android
48     * @param optimizedDirectory directory where optimized dex files
49     *     should be written; must not be {@code null}
50     * @param libraryPath the list of directories containing native
51     *     libraries, delimited by {@code File.pathSeparator}; may be
52     *     {@code null}
53     * @param parent the parent class loader
54     */
55    public DexClassLoader(String dexPath, String optimizedDirectory,
56            String libraryPath, ClassLoader parent) {
57        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
58    }
```

一般我们都是用这个DexClassLoader来作为动态加载的加载器  



 ## 参考
[一看你就懂，超详细java中的ClassLoader详解](http://blog.csdn.net/briblue/article/details/54973413)  
[类加载机制：全盘负责和双亲委托](http://blog.csdn.net/zhangzeyuaaa/article/details/42499839)

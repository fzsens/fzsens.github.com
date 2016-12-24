---
layout: post
title: Java ClassLoader 运行机制
date: 2015-3-20
categories: java
tags: [classloader]
description: 介绍Java ClassLoader运行机制
---

### 什么是ClassLoader

Java类要在程序中使用，需要加载到虚拟机中；执行这一过程的类就是类加载器(`ClassLoader`)。`ClassLoader`一般通过类的全限定名称，从网络(比如`Applet`)、文件系统、内存(由代码生成的类)中加载类的二进制字节流，通过校验和链接等一系列动作之后，将类加载到虚拟机中。每一个在虚拟机中的类都可以找到对应的类加载器，通过`Class.getClassLoader()`来获取得到其类加载器。

一个类不能被重复加载到同一个`ClassLoader`中；同一个类被不同`ClassLoader`加载，在JVM中相当于是两个不同的类。  

````java
DefineClassLoader dcl = new DefineClassLoader();
Class<?> clazz1 = dcl.loadClass("com.qiongsong.classloader.Test.class");

DefineClassLoader dcl2 = new DefineClassLoader();
Class<?> clazz2 = dcl2.loadClass("com.qiongsong.classloader.Test.class");

System.out.println(clazz1 == clazz2);
````

输出的值为 `false`。

### ClassLoader的分类

类加载器可以分成四种类型

- **Boostrap ClassLoader** 引导类加载器(C++实现)，用于加载Java运行时需要的核心类，如`rt.jar`等
- **ExtClassLoader** 加载标准拓展类库，JDK默认的拓展类库路径位于`lib\ext`，将jar包放在此处可以被`ExtClassLoader`加载
- **系统类加载器(AppClassLoader)** 将系统类路径(ClassPath)下的类库加载入内存，可以通过 `ClassLoader.getSystemClassLoader()` 来获取对应的实例，通过`System.getProperty("java.class.path")`来获取加载的文件路径
- **自定义的ClassLoader** 一般通过继承`ClassLoader`，通过自定义`DefineClass`来实现自定义类加载。

### ClassLoader 运行机制

`ClassLoader`在实现的时候使用层级委托的形式，除了引导类加载器之外的每一个`ClassLoader`都具有父`ClassLoader`,如下图：

![classloader继承](https://i.imgur.com/WOiuTUQ.png)

在尝试进行类加载的时候，优先尝试使用父`ClassLoader`进行加载，如果父ClassLoader为空(`ExClassLoader`的父加载器为`Boostrap ClassLoader`，但是`BoostrapClassLoader`为C++实现，无法在Java代码中获取实例，因此`ExtClassLoader`的`parent`为`null`),则尝试`Boostrap ClassLoader`，最后尝试自定义类加载器，可以在`ClassLoader`类中看到对应的实现：

````java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    //获取锁，防止同一个Class被重复加载
    synchronized (getClassLoadingLock(name)) {
        // 校验Class是否已经被加载，在一个ClassLoader中，一个全限定名的类只会被加载一次
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    // 实际上为一个递归，依次调用父类的loadClass方法
                    c = parent.loadClass(name, false);
                } else {
                    // 尝试调用引导类加载器
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {

            }
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                c = findClass(name);
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
````

`ClassLoader`类为类加载器的抽象类，`AppClassLoader` 以及 `ExtClassLoader` 都继承自 `ClassLoader` 类，`loadClass`为调用累加载的入。在`loadClass`方法中进行向上委托而无法顺利加载类的时候就会调用`findClass`方法；这个方法也是官方推荐自定义`ClassLoader`时候实现的方法，在`ClassLoader`中`findClass`为抽象方法，需要在各个实现类中实现。使用逐级委托的调用方式，可以保证Java中的核心类可以正确加载到对应的类加载器中。如 `java.lang.Object`和`java.lang.Object` 等核心类被`Boostrap ClassLoader`加载之后，可以避免再被加载到自定义的`ClassLoader`中，而使得系统中具有多个 `java.lang.Object`，实际上如果使用默认的`defineClass`进行核心类的加载会报出`java.lang.SecurityException:Prohibited package name: java.lang`错误，这也是虚拟机安全机制的一部分。

### 如何自定义ClassLoader

为了保证类加载的符合委托加载机制，官方给出的建议是实现`findClass`方法，实现类全限定名->二进制字节->内存中的Class对象的转换需要在`protected final Class<?> defineClass(String name, byte[] b, int off, int len)`方法中实现,下面我们自定义类加载器从，文件系统中加载Class文件到虚拟机中。

自定义的类加载器，继承`ClassLoader`类，实现`findClass`方法

````java
public byte[] loadClassData(String fileName) {
    File classFile = new File(fileName);
    ByteArrayOutputStream byteArrayOutputStream = null;
    try {
        fileInputStream = new FileInputStream(classFile);
        byte[] buffer = new byte[1024];
        byteArrayOutputStream = new ByteArrayOutputStream();
        int readByteCount = -1;
        while ((readByteCount = fileInputStream.read(buffer)) != -1) {
            byteArrayOutputStream.write(buffer, 0, readByteCount);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return byteArrayOutputStream == null ? null : byteArrayOutputStream
            .toByteArray();
}

@Override
public Class<?> findClass(String qualifiedName)
        throws ClassNotFoundException {
    if (qualifiedName.endsWith(".class")) {
        qualifiedName = qualifiedName.substring(0,
                qualifiedName.lastIndexOf("."));
    }
    String className = qualifiedName.substring(qualifiedName
            .lastIndexOf(".") + 1);
    // 定义文件系统目录为 D盘 
    String fileName = "D:\\" + className + ".class";
    //将文件转换为字节流
    byte[] classBytes = loadClassData(fileName);
    return defineClass(qualifiedName, classBytes, 0, classBytes.length);
}
````

定义两个测试类

````java
public class SuperTest {

    public void dosomethings() {
       System.out.println(" SuperTest ClassLoader " + SuperTest.class.getClassLoader());
    }
}

public class Test extends SuperTest {

    public void sayHello() {
        System.out.println(" Test ClassLoader " + Test.class.getClassLoader());
    }

}
````

并将编译出来的Class文件，放在D盘的根目录中，执行下面的测试方法

````java
public static void main(String[] args) throws Exception {
    DefineClassLoader dcl = new DefineClassLoader();
    // dcl.loadClass("com.qiongsong.classloader.SuperTest.class");
    // 父类的加载依然符合委托代理加载机制
    Class<?> clazz = dcl.loadClass("com.qiongsong.classloader.Test.class");
    Object object = clazz.newInstance();
    Method method = clazz.getMethod("sayHello");
    Method method2 = clazz.getMethod("dosomethings");
    method.invoke(object);
    method2.invoke(object);

    DefineClassLoader dcl2 = new DefineClassLoader();
    Class<?> clazz2 = dcl2
            .loadClass("com.qiongsong.classloader.Test.class");
    Object object2 = clazz2.newInstance();
    Method method3 = clazz2.getMethod("sayHello");
    Method method4 = clazz2.getMethod("dosomethings");
    method3.invoke(object2);
    method4.invoke(object2);

    System.out.println(object.getClass() == object2.getClass());
}
````

输出结果为

Test ClassLoader com.qiongsong.classloader.DefineClassLoader@15db9742 SuperTest ClassLoader  
com.qiongsong.classloader.DefineClassLoader@15db9742 Test ClassLoader  
com.qiongsong.classloader.DefineClassLoader@70dea4e SuperTest ClassLoader  
com.qiongsong.classloader.DefineClassLoader@70dea4e  
false  

`Class<?> forName(String name, boolean initialize, ClassLoader loader)`可以通过类全限定名来获取Class,如果未指定loader参数，默认使用当前调用类的类加载器。`Class.forName(“Foo”)等同于 Class.forName(“Foo”, true, this.getClass().getClassLoader())`

### 类并发加载

为保证同一个类只能在一个ClassLoader中加载一次，需要对loadClass进行同步，在上文中举例的`ClassLoader`的`loadClass`方法为Java1.7以及之后的实现，在Java1.6的实现为

````java
protected synchronized Class<?> loadClass(String paramString, boolean paramBoolean)
    throws ClassNotFoundException
  {
    Class localClass = findLoadedClass(paramString);
    if (localClass == null) {
      try {
        if (this.parent != null)
          localClass = this.parent.loadClass(paramString, false);
        else {
          localClass = findBootstrapClassOrNull(paramString);
        }
      }
      catch (ClassNotFoundException localClassNotFoundException)
      {
      }
      if (localClass == null)
      {
        localClass = findClass(paramString);
      }
    }
    if (paramBoolean) {
      resolveClass(localClass);
    }
    return localClass;
  }
````

在1.6的实现中，直接在`loadClass`方法加上同步原语`synchronized`，同一个`ClassLoader`对象只能实现串行加载，在1.7的实现为在`synchronized (getClassLoadingLock(name))，getClassLoadingLock`的代码实现为：

````java
protected Object getClassLoadingLock(String className) {
    Object lock = this;
    if (parallelLockMap != null) {
        Object newLock = new Object();
        lock = parallelLockMap.putIfAbsent(className, newLock);
        if (lock == null) {
            lock = newLock;
        }
    }
    return lock;
}
````

其中`parallelLockMap`在`ParallelLoaders`中注册为允许并行加载的时候，会进行初始化 `parallelLockMap = new ConcurrentHashMap<>()`，在`getClassLoadingLock`中为每一个类名设置一个锁对象，每一个相同类名的类在加载到同一个`ClassLoader`的时候，会获取同一个锁，从而实现达到了同步的效果。通过这样的设置，可以实现类加载器对不同类记性并行加载。对比1.6的对象锁而言性能有所提升。
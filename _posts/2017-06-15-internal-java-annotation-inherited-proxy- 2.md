---
layout: post
title: Annotation和动态代理-2
date: 2017-06-14
categories: java
tags: [java]
description: 动态代理中Annotation的继承管理和动态代理原理
---

为了方便表述，下面分不同的章节分别描述不同代理类的差异，最后统一进行比较

#### JDK Proxy

从上一篇分析中，我们已经可以得到，JDK Proxy生成的代理类为Proxy0，利用HSDB，从classes browser中可以将.class文件导出，通过procyon来进行反编译，得到内容如下

````java

package com.sun.proxy;

import com.qiongsong.dynamic.ProxyTester.IProxyMe;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0
  extends Proxy
  implements ProxyTester.IProxyMe
{
  private static Method m1;
  private static Method m3;
  private static Method m2;
  private static Method m0;
  //初始化
  public $Proxy0(InvocationHandler paramInvocationHandler)
  {
    super(paramInvocationHandler);
  }
  
  static
  {
    try
    {
	  // 静态代码块，通过放射得到方法
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("com.qiongsong.dynamic.ProxyTester$IProxyMe").getMethod("aMethod", new Class[] { Integer.TYPE });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
  
  public final boolean equals(Object paramObject)
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
  
  public final String toString()
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
  
  public final int hashCode()
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
  
  public final void aMethod(int paramInt)
  {
    try
    {
	  // 使用代理类执行的时候，委托给h进行执行
      this.h.invoke(this, m3, new Object[] { Integer.valueOf(paramInt) });
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
}

````

从生成的代码中，可以很清楚看到

1. JDK proxy生成的代码是面向接口的
2. 静态代码块中获取对应**IProxyMe接口**的方法，定义为类变量
3. 初始化的时候，会传入`InvocationHandler`实例，并使用父构造函数进行初始化，实际上赋值为`java.lang.reflection.Proxy`中的`protected InvocationHandler h;`变量
4. 进行aMethod方法调用的时候，第一个参数为自己，第二个参数为静态代码块中构建的method方法，这个方法的实际拥有者是接口方法，第三个参数是将入参包装为Object[]

再看一下`java.lang.reflect.Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)`接口定义，就很容易将JDK的动态代理过程理解清楚，同时当将`IProxyMe`的Method和Parameter Annotation去除之后，在新输出的中jdk proxy对应的

````java

<<<<< ---- Invoking methods ----- >>>>>
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe_$$_jvst813_0
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 1
-----------------
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 2
-----------------
Type: class com.sun.proxy.$Proxy0
     Has 0  type annotation(s)
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
Invoked 3
-----------------
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$ByteBuddy$NI8f8o5m
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 4
-----------------

````


Method调用过程中也无法得到Annotation，因为其Method是从`IProxyMe`接口中获取的，而其他的代理对象，则依然可以正常获取到Annotation，可知在这块实现上各个动态代理框架的实现过程是不一样的。而`Class<?>[] interfaces`参数也限制了jdk proxy必须建立在接口的基础上，原因也很简单，当你extends Proxy之后，无法多重继承的Java，显然没有办法再继承代理实现类。

#### Cglib

也正是因为jdk proxy的必须基于接口这个特点的限制，在没有办法确定所有的对象都有接口定义的情况下，我们会选择cglib来作为动态代理框架，cglib底层采用ASM字节码生成框架，使用字节码技术生成代理类，比使用jdk proxy的执行效率更高，但是生成会稍微慢一点。使用同样的方法提取.class，使用cglib生成的的内容会比较复杂，这边不对cglib的生成原理进行介绍，主要通过生成的字节码文件，来说明整个动态代理调用的过程。

​````java
// 不同jdk proxy，cglib使用了继承实现类，并实现接口的方式，因此cglib没有必须使用接口的限制，
// 使用继承的方式，超类中标记为final的方法无法被override，因此cglib无法代理生成final方法
public class ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c extends ProxyTester$ProxyMe implements ProxyTester$IProxyMe, Factory
{
    private boolean CGLIB$BOUND;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
	// 可以发现每一个方法，在cglib内部都会对应一个Method和一个MethodProxy，
	// 这个MethodProxy，就是在MethodInterceptor中入口使用的MethodProxy，一般用于调用委托类(一般也就是超类)的原方法（invokeSuper），或者调用代理类(就是本类)的方法(invoke)
	// 为此MethodProxy，会生成两个独立的FastCglib代理类
    private static final Method CGLIB$aMethod$0$Method;
    private static final MethodProxy CGLIB$aMethod$0$Proxy;

    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$finalize$1$Method;
    private static final MethodProxy CGLIB$finalize$1$Proxy;
    private static final Method CGLIB$equals$2$Method;
    private static final MethodProxy CGLIB$equals$2$Proxy;
    private static final Method CGLIB$toString$3$Method;
    private static final MethodProxy CGLIB$toString$3$Proxy;
    private static final Method CGLIB$hashCode$4$Method;
    private static final MethodProxy CGLIB$hashCode$4$Proxy;
    private static final Method CGLIB$clone$5$Method;
    private static final MethodProxy CGLIB$clone$5$Proxy;

    static {
        CGLIB$STATICHOOK1();
    }

	// 静态代码块，在class 加载到classloader的时候，被调用
    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        final Class<?> forName = Class.forName("com.qiongsong.dynamic.ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c");
        final Class<?> forName2;
		// method 和 methodproxy生成
        CGLIB$aMethod$0$Method = ReflectUtils.findMethods(new String[] { "aMethod", "(I)V" }, (forName2 = Class.forName("com.qiongsong.dynamic.ProxyTester$ProxyMe")).getDeclaredMethods())[0];
        CGLIB$aMethod$0$Proxy = MethodProxy.create((Class)forName2, (Class)forName, "(I)V", "aMethod", "CGLIB$aMethod$0");

        final Class<?> forName3;
        final Method[] methods = ReflectUtils.findMethods(new String[] { "finalize", "()V", "equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;" }, (forName3 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$finalize$1$Method = methods[0];
        CGLIB$finalize$1$Proxy = MethodProxy.create((Class)forName3, (Class)forName, "()V", "finalize", "CGLIB$finalize$1");
        CGLIB$equals$2$Method = methods[1];
        CGLIB$equals$2$Proxy = MethodProxy.create((Class)forName3, (Class)forName, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$2");
        CGLIB$toString$3$Method = methods[2];
        CGLIB$toString$3$Proxy = MethodProxy.create((Class)forName3, (Class)forName, "()Ljava/lang/String;", "toString", "CGLIB$toString$3");
        CGLIB$hashCode$4$Method = methods[3];
        CGLIB$hashCode$4$Proxy = MethodProxy.create((Class)forName3, (Class)forName, "()I", "hashCode", "CGLIB$hashCode$4");
        CGLIB$clone$5$Method = methods[4];
        CGLIB$clone$5$Proxy = MethodProxy.create((Class)forName3, (Class)forName, "()Ljava/lang/Object;", "clone", "CGLIB$clone$5");
    }

	// 直接调用原始方法
    final void CGLIB$aMethod$0(final int n) {
        super.aMethod(n);
    }

	// 代理方法，当我们使用代理类的时候，会调用这个方法
    public final void aMethod(final int n) {
        MethodInterceptor cglib$CALLBACK_2;
        MethodInterceptor cglib$CALLBACK_0;
		// 绑定MethodInterceptor
        if ((cglib$CALLBACK_0 = (cglib$CALLBACK_2 = this.CGLIB$CALLBACK_0)) == null) {
            CGLIB$BIND_CALLBACKS(this);
            cglib$CALLBACK_2 = (cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0);
        }
        if (cglib$CALLBACK_0 != null) {
			// 执行MethodInterceptor的方法，这个方法也是我们在创建cglib的时候，传入的匿名类，
			// 同样的第一个参数是 代理类自己，第二个参数，是利用放射从接口中获取到的Method，第三个参数，是封装成Object[]的参数，第四个参数，是刚才降到的MethodProxy
            cglib$CALLBACK_2.intercept((Object)this, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$aMethod$0$Method, new Object[] { new Integer(n) }, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$aMethod$0$Proxy);
            return;
        }
        super.aMethod(n);
    }

    final void CGLIB$finalize$1() throws Throwable {
        super.finalize();
    }

    protected final void finalize() throws Throwable {
        MethodInterceptor cglib$CALLBACK_2;
        MethodInterceptor cglib$CALLBACK_0;
        if ((cglib$CALLBACK_0 = (cglib$CALLBACK_2 = this.CGLIB$CALLBACK_0)) == null) {
            CGLIB$BIND_CALLBACKS(this);
            cglib$CALLBACK_2 = (cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0);
        }
        if (cglib$CALLBACK_0 != null) {
            cglib$CALLBACK_2.intercept((Object)this, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$finalize$1$Method, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$emptyArgs, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$finalize$1$Proxy);
            return;
        }
        super.finalize();
    }

    final boolean CGLIB$equals$2(final Object o) {
        return super.equals(o);
    }

    public final boolean equals(final Object o) {
        MethodInterceptor cglib$CALLBACK_2;
        MethodInterceptor cglib$CALLBACK_0;
        if ((cglib$CALLBACK_0 = (cglib$CALLBACK_2 = this.CGLIB$CALLBACK_0)) == null) {
            CGLIB$BIND_CALLBACKS(this);
            cglib$CALLBACK_2 = (cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0);
        }
        if (cglib$CALLBACK_0 != null) {
            final Object intercept = cglib$CALLBACK_2.intercept((Object)this, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$equals$2$Method, new Object[] { o }, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$equals$2$Proxy);
            return intercept != null && (boolean)intercept;
        }
        return super.equals(o);
    }

    final String CGLIB$toString$3() {
        return super.toString();
    }

    public final String toString() {
        MethodInterceptor cglib$CALLBACK_2;
        MethodInterceptor cglib$CALLBACK_0;
        if ((cglib$CALLBACK_0 = (cglib$CALLBACK_2 = this.CGLIB$CALLBACK_0)) == null) {
            CGLIB$BIND_CALLBACKS(this);
            cglib$CALLBACK_2 = (cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0);
        }
        if (cglib$CALLBACK_0 != null) {
            return (String)cglib$CALLBACK_2.intercept((Object)this, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$toString$3$Method, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$emptyArgs, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$toString$3$Proxy);
        }
        return super.toString();
    }

    final int CGLIB$hashCode$4() {
        return super.hashCode();
    }

    public final int hashCode() {
        MethodInterceptor cglib$CALLBACK_2;
        MethodInterceptor cglib$CALLBACK_0;
        if ((cglib$CALLBACK_0 = (cglib$CALLBACK_2 = this.CGLIB$CALLBACK_0)) == null) {
            CGLIB$BIND_CALLBACKS(this);
            cglib$CALLBACK_2 = (cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0);
        }
        if (cglib$CALLBACK_0 != null) {
            final Object intercept = cglib$CALLBACK_2.intercept((Object)this, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$hashCode$4$Method, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$emptyArgs, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$hashCode$4$Proxy);
            return (intercept == null) ? 0 : ((Number)intercept).intValue();
        }
        return super.hashCode();
    }

    final Object CGLIB$clone$5() throws CloneNotSupportedException {
        return super.clone();
    }

    protected final Object clone() throws CloneNotSupportedException {
        MethodInterceptor cglib$CALLBACK_2;
        MethodInterceptor cglib$CALLBACK_0;
        if ((cglib$CALLBACK_0 = (cglib$CALLBACK_2 = this.CGLIB$CALLBACK_0)) == null) {
            CGLIB$BIND_CALLBACKS(this);
            cglib$CALLBACK_2 = (cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0);
        }
        if (cglib$CALLBACK_0 != null) {
            return cglib$CALLBACK_2.intercept((Object)this, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$clone$5$Method, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$emptyArgs, ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$clone$5$Proxy);
        }
        return super.clone();
    }

    public static MethodProxy CGLIB$findMethodProxy(final Signature signature) {
        final String string = signature.toString();
        switch (string.hashCode()) {
            case -1574182249: {
                if (string.equals("finalize()V")) {
                    return ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$finalize$1$Proxy;
                }
                break;
            }
            case -1236445744: {
                if (string.equals("aMethod(I)V")) {
                    return ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$aMethod$0$Proxy;
                }
                break;
            }
            case -508378822: {
                if (string.equals("clone()Ljava/lang/Object;")) {
                    return ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$clone$5$Proxy;
                }
                break;
            }
            case 1826985398: {
                if (string.equals("equals(Ljava/lang/Object;)Z")) {
                    return ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$equals$2$Proxy;
                }
                break;
            }
            case 1913648695: {
                if (string.equals("toString()Ljava/lang/String;")) {
                    return ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$toString$3$Proxy;
                }
                break;
            }
            case 1984935277: {
                if (string.equals("hashCode()I")) {
                    return ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$hashCode$4$Proxy;
                }
                break;
            }
        }
        return null;
    }

	// 构造方法
    public ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c() {
        CGLIB$BIND_CALLBACKS(this);
    }

    public static void CGLIB$SET_THREAD_CALLBACKS(final Callback[] array) {
        ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$THREAD_CALLBACKS.set(array);
    }

    public static void CGLIB$SET_STATIC_CALLBACKS(final Callback[] cglib$STATIC_CALLBACKS) {
        CGLIB$STATIC_CALLBACKS = cglib$STATIC_CALLBACKS;
    }

    private static final void CGLIB$BIND_CALLBACKS(final Object o) {
        final ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c = (ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c)o;
        if (!proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$BOUND) {
            proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$BOUND = true;
            Object o2;
            if ((o2 = ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$THREAD_CALLBACKS.get()) != null || (o2 = ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$STATIC_CALLBACKS) != null) {
                proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])o2)[0];
            }
        }
    }

    public Object newInstance(final Callback[] array) {
        CGLIB$SET_THREAD_CALLBACKS(array);
        final ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c = new ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c();
        CGLIB$SET_THREAD_CALLBACKS(null);
        return proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c;
    }

    public Object newInstance(final Callback callback) {
        CGLIB$SET_THREAD_CALLBACKS(new Callback[] { callback });
        final ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c = new ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c();
        CGLIB$SET_THREAD_CALLBACKS(null);
        return proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c;
    }

    public Object newInstance(final Class[] array, final Object[] array2, final Callback[] array3) {
        CGLIB$SET_THREAD_CALLBACKS(array3);
        switch (array.length) {
            case 0: {
                final ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c = new ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c();
                CGLIB$SET_THREAD_CALLBACKS(null);
                return proxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c;
            }
            default: {
                throw new IllegalArgumentException("Constructor not found");
            }
        }
    }

    public Callback getCallback(final int n) {
        CGLIB$BIND_CALLBACKS(this);
        Object cglib$CALLBACK_0 = null;
        switch (n) {
            case 0: {
                cglib$CALLBACK_0 = this.CGLIB$CALLBACK_0;
                break;
            }
            default: {
                cglib$CALLBACK_0 = null;
                break;
            }
        }
        return (Callback)cglib$CALLBACK_0;
    }

    public void setCallback(final int n, final Callback callback) {
        switch (n) {
            case 0: {
                this.CGLIB$CALLBACK_0 = (MethodInterceptor)callback;
                break;
            }
        }
    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[] { this.CGLIB$CALLBACK_0 };
    }

    public void setCallbacks(final Callback[] array) {
        this.CGLIB$CALLBACK_0 = (MethodInterceptor)array[0];
    }

}


​````

从整体上讲，cglib的整个代理过程还是和jdk proxy比较接近的，利用高级的字节码技术，使得cglib比使用放射技术实现的jdk proxy更加灵活，在Annotation相关的操作中，因为使用了继承机制，因此cglib的类可以顺利继承TypeAnnotation，在Method和Parameter Annotation方面，则两者基本一样。在使用HSDB查看类的时候，会发现有三个类，其中`ProxyTester$ProxyMe$$FastClassByCGLIB$$c2d280aa.class` 和`ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c$$FastClassByCGLIB$$67b37e23.class`为`MethodProxy`自动生成的FastCglib类，前者指向代理类，后者指向委托类，也就是说，如果MethodProxy调用`invokeSuper`，第二个参数，需要是代理类，则会直接`ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c$$FastClassByCGLIB$$67b37e23.class`中的CGLIB$aMethod$0(final int n)，而如果调用`invoke`则会根据传入的参数，调用对应的`aMethod`方法，如果直接调用代理类，显然会导致`StackOverFlow`的错误。

> FastCglib的内部实现，是使用下标来标记方法，当进行调用的时候，通过方法签名获取下标，直接通过下标做直接调用，省去了反射的消耗

#### javassist

javassist是另一款动态字节码生成类库，它的主要特点是提供Source级别的API，可以编码的方式来生成或者修改类，而不需要对字节码有太多了解，当然也支持低级别bytecode级别的API，使用bytecode级别的api效率更高。

​````java
//javassist的类签名和cglib类似，使用了继承的方式
public class ProxyTester$ProxyMe_$$_jvst5ab_0 extends ProxyTester$ProxyMe implements ProxyObject
{
    private MethodHandler handler;
    public static byte[] _filter_signature;
    public static final long serialVersionUID;
	// 存放 Method
    private static Method[] _methods_;

    public MethodHandler getHandler() {
        return this.handler;
    }

    public ProxyTester$ProxyMe_$$_jvst5ab_0() {
        this.handler = RuntimeSupport.default_interceptor;
    }

    static throws ClassNotFoundException {
        final Method[] methods_ = new Method[24];
        final Class<?> forName = Class.forName("com.qiongsong.dynamic.ProxyTester$ProxyMe_$$_jvst5ab_0");
		// 构建方法数组
        RuntimeSupport.find2Methods((Class)forName, "aMethod", "_d0aMethod", 0, "(I)V", methods_);
        RuntimeSupport.find2Methods((Class)forName, "clone", "_d1clone", 2, "()Ljava/lang/Object;", methods_);
        RuntimeSupport.find2Methods((Class)forName, "equals", "_d2equals", 4, "(Ljava/lang/Object;)Z", methods_);
        RuntimeSupport.find2Methods((Class)forName, "finalize", "_d3finalize", 6, "()V", methods_);
        RuntimeSupport.find2Methods((Class)forName, "hashCode", "_d5hashCode", 10, "()I", methods_);
        RuntimeSupport.find2Methods((Class)forName, "toString", "_d8toString", 16, "()Ljava/lang/String;", methods_);
        ProxyTester$ProxyMe_$$_jvst5ab_0._methods_ = methods_;
        serialVersionUID = 4294967295L;
    }

    protected final void finalize() throws Throwable {
        final Method[] methods_ = ProxyTester$ProxyMe_$$_jvst5ab_0._methods_;
        this.handler.invoke((Object)this, methods_[6], methods_[7], new Object[0]);
    }

    public final boolean equals(final Object o) {
        final Method[] methods_ = ProxyTester$ProxyMe_$$_jvst5ab_0._methods_;
        return (boolean)this.handler.invoke((Object)this, methods_[4], methods_[5], new Object[] { o });
    }

    public final String toString() {
        final Method[] methods_ = ProxyTester$ProxyMe_$$_jvst5ab_0._methods_;
        return (String)this.handler.invoke((Object)this, methods_[16], methods_[17], new Object[0]);
    }

    public final int hashCode() {
        final Method[] methods_ = ProxyTester$ProxyMe_$$_jvst5ab_0._methods_;
        return (int)this.handler.invoke((Object)this, methods_[10], methods_[11], new Object[0]);
    }

    protected final Object clone() throws CloneNotSupportedException {
        final Method[] methods_ = ProxyTester$ProxyMe_$$_jvst5ab_0._methods_;
        return this.handler.invoke((Object)this, methods_[2], methods_[3], new Object[0]);
    }

    Object writeReplace() throws ObjectStreamException {
        return RuntimeSupport.makeSerializedProxy((Object)this);
    }

    public final void aMethod(final int n) {
        final Method[] methods_ = ProxyTester$ProxyMe_$$_jvst5ab_0._methods_;
		// 调用的方法参数，第一个参数为代理类，第二个参数为本方法，第三个参数为原方法也就是_d0aMethod，第四个参数为构造成Object[]的参数
        this.handler.invoke((Object)this, methods_[0], methods_[1], new Object[] { new Integer(n) });
    }

    public void setHandler(final MethodHandler handler) {
        this.handler = handler;
    }

    public final void _d0aMethod(final int n) {
        super.aMethod(n);
    }

    public final Object _d1clone() throws CloneNotSupportedException {
        return super.clone();
    }

    public final boolean _d2equals(final Object o) {
        return super.equals(o);
    }

    public final void _d3finalize() throws Throwable {
        super.finalize();
    }

    public final int _d5hashCode() {
        return super.hashCode();
    }

    public final String _d8toString() {
        return super.toString();
    }
}

​````

javassist的实现相比cglib要简单一些，`RuntimeSupport.find2Methods((Class)forName, "aMethod", "_d0aMethod", 0, "(I)V", methods_);`构造方法数组，

​````java

public static void find2Methods(Class clazz, String superMethod,
                                String thisMethod, int index,
                                String desc, java.lang.reflect.Method[] methods)
{
    methods[index + 1] = thisMethod == null ? null
                                            : findMethod(clazz, thisMethod, desc);
    methods[index] = findSuperClassMethod(clazz, superMethod, desc);
}

​````

下标index为代理类的方法，下标为index+1位继承类的对应方法，如果方法没有定义，例如接口或者abstract class ，则index+1为null。

#### ByteBuddy

ByteBuddy的功能和cglib类似，在官方提供的benchmark测试中，性能优于cglib等其他的工具

​````java

public class ProxyTester$ProxyMe$ByteBuddy$s1Pgctn8 extends ProxyTester$ProxyMe
{
	// 委托方法
    public static /* synthetic */ ProxyTester$1MyInterceptor delegate$5k4gee1;
    private static final /* synthetic */ Method cachedValue$3isHCYP0$fgups02;

    static {
		// 原方法
        cachedValue$3isHCYP0$fgups02 = ProxyTester$ProxyMe.class.getDeclaredMethod("aMethod", Integer.TYPE);
    }

    public void aMethod(final int n) {
		// 第一个参数为代理类，第二个参数为原方法，第三个参数为包装为Object的参数
        ProxyTester$ProxyMe$ByteBuddy$s1Pgctn8.delegate$5k4gee1.intercept((Object)this, ProxyTester$ProxyMe$ByteBuddy$s1Pgctn8.cachedValue$3isHCYP0$fgups02, new Object[] { n });
    }
}

​````

ByteBuddy的生成类最为简洁，很容易就可以看得懂。

#### 总结

动态代理的实现方式中ASM过于复杂，cglib和javassist是一般情况下是我使用动态代理的首选，jdk proxy使用了反射API，效率相比直接使用字节码较低，byteBuddy是刚刚接触了解不多。原本只是简单的Annotation继承问题，最后变成对动态代理工具的分析，稍微有点偏题了。
```
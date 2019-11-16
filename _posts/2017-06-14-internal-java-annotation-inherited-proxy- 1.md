---
layout: post
title: Annotation和动态代理-1
date: 2017-06-14
categories: java
tags: [java]
description: 动态代理中Annotation的继承管理和动态代理原理
---

今天在集成规则引擎代码的时候，发现一个(些)有趣的问题

#### 类型注解为何频频丢失？

我们的规则引擎通过`Annotation`实现流程分支和行为的控制，当我将规则使用`@Service`托管给Spring之后，却出现无法通过引擎规则验证的情况的情况，经过简单的排除，发现是因为通过Spring注入的实例获取`Annotation`的时候无法正常获取到对应的注解。

```java

A foundAnnotation = annotatedType.getAnnotation(targetAnnotation);

```
但是实现的RuleImpl确实有写注解的，分析得到，原来通过Spring拓展的Rule实例，是由通过cglib动态生成的代理类，而cglib的代理，继承了代理类，在查看注解类型之后，原来是因为规则注解`@Rule`，没有设置`@Inherited` meta-annotation，添加`Inherited`的注解，在通过`getAnnotation(AnnotationClass)`获取注解的时候，如果在当前类获取不到，则会到超类进行查找直到找到，或者到Object为止，代码实现如下

```java

private AnnotationData createAnnotationData(int classRedefinedCount) {
    Map<Class<? extends Annotation>, Annotation> declaredAnnotations =
        AnnotationParser.parseAnnotations(getRawAnnotations(), getConstantPool(), this);
	// 获取超类
    Class<?> superClass = getSuperclass();
    Map<Class<? extends Annotation>, Annotation> annotations = null;
    if (superClass != null) {
		// 超类的注解
        Map<Class<? extends Annotation>, Annotation> superAnnotations =
            superClass.annotationData().annotations;
        for (Map.Entry<Class<? extends Annotation>, Annotation> e : superAnnotations.entrySet()) {
            Class<? extends Annotation> annotationClass = e.getKey();
			// 带有@Inherited元注解的注解
            if (AnnotationType.getInstance(annotationClass).isInherited()) {
                if (annotations == null) { // lazy construction
                    annotations = new LinkedHashMap<>((Math.max(
                            declaredAnnotations.size(),
                            Math.min(12, declaredAnnotations.size() + superAnnotations.size())
                        ) * 4 + 2) / 3
                    );
                }
                annotations.put(annotationClass, e.getValue());
            }
        }
    }
    if (annotations == null) {
        // no inherited annotations -> share the Map with declaredAnnotations
        annotations = declaredAnnotations;
    } else {
        // at least one inherited annotation -> declared may override inherited
        annotations.putAll(declaredAnnotations);
    }
    return new AnnotationData(annotations, declaredAnnotations, classRedefinedCount);
}

```

解决的方法有两种，一种是在`@Rule`中增加`@Inherited`，另一种是自己手工编写获取注解的逻辑，具体可以参考Spring中AnnotationUtils.findXXX，系列方法，原理也和上面的类似

```java

	for (Class<?> ifc : clazz.getInterfaces()) {
		A annotation = findAnnotation(ifc, annotationType, visited);
		if (annotation != null) {
			return annotation;
		}
	}

	Class<?> superclass = clazz.getSuperclass();
	if (superclass == null || Object.class == superclass) {
		return null;
	}
	// recursive call
	return findAnnotation(superclass, annotationType, visited);

```

当找不到对应的注解时，进行一次递归调用。

#### 方法注解为何屡屡失效？

在处理了上面的类型注解丢失问题之后，问题依然存在，规则引擎中除了Type Annotation之外，还有Method Annotation以及Parameter Annotation，原本以为为每一个注解增加`@Inherited`之后可以解决，查看`@Inherited`文档发现了下面的描述

> Note that this meta-annotation type has no effect if the annotated type is used to annotate anything other than a class.  Note also that this meta-annotation only causes annotations to be inherited from superclasses; annotations on implemented interfaces have no effect.
> 注意，这个meta-annotation 类型对于除了class类型之外的注解无效，并且注解只会通过超类得到继承，标记在接口上的不会生效

从上面的`createAnnotationData()`代码来看，也确实是如此，因此获取方法注解的部分，需要通过自行遍历超类来获取。

#### 这一切究竟是注解的扭曲还是动态代理的沦丧？

通过上面的分析，我们可以知道代理类会继承原来的实现类/实现接口，从代理类的功能上来看，主要是在方法调用的前后织入一些固定的逻辑，从而影响实现类的行为。那么不同的动态代理实现方案是否会存在区别，或者说不同的动态代理，最终的代理类型究竟是什么样子？另外在处理方法注解丢失问题的时候，发现使用jdk代理类型的`MethodInterceptor`中的`invoke(Object proxy, Method method, Object[] args)`方法中，`Method`却可以正常获取到Method Annotation，可知这个对象是肯定不是由代理类所拥有，而是由被代理类，或者被代理接口所拥有。那是不是说有的动态代理都是使用类似这样的结构？自然引发了我的一点好奇心，趁这个机会也好了解一下动态代理生成的内容

首先定义三种不同类型的注解

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public static @interface TypeAnnotation {

}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public static @interface MehtodAnnotation {

}


@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public static @interface ParamAnnotation {
}

```

因为我们主要研究的是运行时注解，因此保留策略是RUNTIME，目标类型分别是TYPE代表类/接口，METHOD代表方法，PARAMETER代表参数注解。

接着定义一对方法和实现类

```java
    @TypeAnnotation
    public static interface IProxyMe {
        @MehtodAnnotation
        void aMethod(@ParamAnnotation int param);
    }

    @TypeAnnotation
    public static class ProxyMe implements IProxyMe {
        @MehtodAnnotation
        public void aMethod(@ParamAnnotation int param) {
            System.out.println("Invoked " + param);
            System.out.println("-----------------");
        }
    }
```

内容很简单，不做赘述。

定义分离各类注解的方法，为了简单起见，将内容直接打印出来

```java

/**
 * @param type 代理类型
 * @param proxy 代理类
 * @param clazz 接口类型
 * @param forMethod 方法名
 */
static void dumpAnnotations(String type, Object proxy, Class<?> clazz, String forMethod) {
    System.out.println(type + " proxy for Class: " + clazz);
    printTypeAnnotations(proxy.getClass());
    for (Method method : proxy.getClass().getMethods()) {
        if (method.getName().equals(forMethod)) {
            printMethodAnnotations(method);
        }
    }
}

static void printTypeAnnotations(Class<?> type) {
    System.out.println("Type: " + type);
    Annotation[] annotations = type.getAnnotations();
    //        Annotation[] annotations = new Annotation[]{
    //                AnnotationUtils.findAnnotation(type,TypeAnnotation.class)
    //        };
    System.out.println("     Has " + annotations.length + "  type annotation(s)");
    for (Annotation annotation : annotations) {
        System.out.println("    Annotation " + annotation.toString());
    }
}

static void printMethodAnnotations(Method method) {
    int paramCount = method.getParameterTypes().length;

    System.out.println("Method: " + method.getName());
    Annotation[] annotations = method.getDeclaredAnnotations();
    System.out.println("     Has " + annotations.length + "  method annotation(s)");
    for (Annotation annotation : annotations) {
        System.out.println("    Annotation " + annotation.toString());
    }

    System.out.println("    Method: " + method.getName() + " has " + paramCount + " parameters");
    for (Annotation[] paramAnnotations : method.getParameterAnnotations()) {
        System.out.println("        Annotations: " + paramAnnotations.length);

        for (Annotation annotation : paramAnnotations) {
            System.out.println("            Annotation " + annotation.toString());
        }
    }
}

```

利用放射方法，获取注解，接着定义不同类型的代理类，这里选择几种最为常用的动态代理框架，jdk，cglib，javassist，byteBuddy

```java

static Object javassistProxy(IProxyMe in) throws Exception {
    ProxyFactory pf = new ProxyFactory();
    pf.setSuperclass(in.getClass());

    MethodHandler handler = new MethodHandler() {
        public Object invoke(Object self, Method thisMethod, Method proceed, Object[] args) throws Throwable {
            printTypeAnnotations(self.getClass());
            if (thisMethod.getName().endsWith("aMethod"))
                printMethodAnnotations(thisMethod);

            return proceed.invoke(self, args);
        }
    };
    return pf.create(new Class<?>[0], new Object[0], handler);
}

static Object cglibProxy(IProxyMe in) throws Exception {
    Object p2 = Enhancer.create(in.getClass(), in.getClass().getInterfaces(),
            new MethodInterceptor() {
                public Object intercept(Object arg0, Method arg1, Object[] arg2, MethodProxy arg3) throws Throwable {
                    printTypeAnnotations(arg0.getClass());
                    printMethodAnnotations(arg1);
                    return arg3.invokeSuper(arg0, arg2);
                }
            });

    return p2;
}

static Object jdkProxy(final IProxyMe in) throws Exception {
    return java.lang.reflect.Proxy.newProxyInstance(in.getClass().getClassLoader(), in.getClass().getInterfaces(),
            new InvocationHandler() {
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    printTypeAnnotations(proxy.getClass());
                    printMethodAnnotations(method);
                    return method.invoke(in, args);
                }
            });
}

static Object byteBuddyProxy(final IProxyMe in) throws IllegalAccessException, InstantiationException {
    class MyInterceptor {
        @RuntimeType
        public void invoke(@This Object target, @Origin Method method, @AllArguments Object[] arguments)
                throws Exception {
            printTypeAnnotations(target.getClass());
            printMethodAnnotations(method);
            method.invoke(in,arguments);
        }
    }
    return new ByteBuddy().subclass(ProxyMe.class).method(ElementMatchers.named("aMethod")).intercept(MethodDelegation.to(new MyInterceptor()))
            .make().load(ProxyMe.class.getClassLoader()).getLoaded().newInstance();
}

```

每一个动态代理在调用aMethod之前都会植入，打印TypeAnnotation和MethodAnnotation的方法。各个动态代理类的写法，可以参考各个文档，一般我们使用较多的为jdk和cglib，byteBuddy在javaagent使用的较多。

最后编写测试类

````java

IProxyMe proxyMe = new ProxyMe();
IProxyMe x = (IProxyMe) javassistProxy(proxyMe);
IProxyMe y = (IProxyMe) cglibProxy(proxyMe);
IProxyMe z = (IProxyMe) jdkProxy(proxyMe);
IProxyMe q = (IProxyMe) byteBuddyProxy(proxyMe);

dumpAnnotations("no", proxyMe, IProxyMe.class, "aMethod");
dumpAnnotations("javassist", x, IProxyMe.class, "aMethod");
dumpAnnotations("cglib", y, IProxyMe.class, "aMethod");
dumpAnnotations("jdk", z, IProxyMe.class, "aMethod");
dumpAnnotations("byteBuddy", q, IProxyMe.class, "aMethod");

System.out.println("<<<<< ---- Invoking methods ----- >>>>>");

x.aMethod(1);
y.aMethod(2);
z.aMethod(3);
q.aMethod(4);

System.in.read();

````

测试的内容也很简单，首先构建实例和代理类实力，接着对代理累进行Annotation分析，然后调用代理方法，在代理方法的植入代码中，再次调用Annotation分析，为了能够提取动态生成的.class文件，在最后加入一个堵塞操作。

> 在提取class文件的时候，我们会用到HSDB(Hostspot Debugger)，我们这里主要会用到，classes browser 的功能，使用Windows7 X64，在Cmd下使用，`Java -classpath "%JAVA_HOME%/lib/sa-jdi.jar" sun.jvm.hotspot.HSDB` 打开HSDB，Attach对应的Java执行进程之后，选择Tools即可看到，更加详细的使用可以参考[借HSDB来探索HotSpot VM的运行时数据](http://rednaxelafx.iteye.com/blog/1847971)

#### 走进科学

在以上默认的情况下，我们会得到下面的输出，为了方便阅读，我直接将分析，写在输出项目中

````java

no proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
**直接使用new 出来的ProxyMe对象，可以正常分离出所有的Annotation**

javassist proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe_$$_jvst813_0
     Has 0  type annotation(s)
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
**使用javassist生成的动态代理类ProxyTester$ProxyMe_$$_jvst813_0，无法分离出Annotation**

cglib proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c
     Has 0  type annotation(s)
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
jdk proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.sun.proxy.$Proxy0
     Has 0  type annotation(s)
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
byteBuddy proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$ByteBuddy$xFn3ZhFb
     Has 0  type annotation(s)
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
**基本情况和javassist一致**

<<<<< ---- Invoking methods ----- >>>>>
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe_$$_jvst813_0
     Has 0  type annotation(s)
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 1
-----------------
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c
     Has 0  type annotation(s)
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
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 3
-----------------
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$ByteBuddy$xFn3ZhFb
     Has 0  type annotation(s)
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 4
**调用时，代理类都无法获得Annotation，而方法拦截器中的调用，可以正常获得Annotation**

````

为了验证`@Inherited`注解的效果，将所有的注解类型都添加`@Inheried`,再看一下输出结果

```shell

no proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
javassist proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe_$$_jvst813_0
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
cglib proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$$EnhancerByCGLIB$$9b74e96c
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
jdk proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.sun.proxy.$Proxy0
     Has 0  type annotation(s)
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0
**jdk proxy 依然无法获得Type Annotation**
byteBuddy proxy for Class: interface com.qiongsong.dynamic.ProxyTester$IProxyMe
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$ByteBuddy$Al0jtkeg
     Has 1  type annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$TypeAnnotation()
Method: aMethod
     Has 0  method annotation(s)
    Method: aMethod has 1 parameters
        Annotations: 0

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
     Has 1  method annotation(s)
    Annotation @com.qiongsong.dynamic.ProxyTester$MehtodAnnotation()
    Method: aMethod has 1 parameters
        Annotations: 1
            Annotation @com.qiongsong.dynamic.ProxyTester$ParamAnnotation()
Invoked 3
-----------------
Type: class com.qiongsong.dynamic.ProxyTester$ProxyMe$ByteBuddy$Al0jtkeg
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

```

比较两次的输出可以发现，添加了`@Inherited`之后，除了jdk proxy之外，其他的代理类型，在两次输出中都可以找到TypeAnnotation，但是Method和Parameter Annotation，依然无法正常获取，通过这个现象初步判断，JDK生成的代理类型，并没有拓展(extends)，我们的实现类`ProxyMe`，而其他的代理对象则extends了ProxyMe。下一篇我们通过HSDB，详细研究一下动态代理生成的.class内容，解开动态代理的面纱。

---
layout: post
title: Spring应用架构（二）核心容器原理和实现
date: 2018-08-31
categories: spring
tags: [spring,java]
description: Spring应用架构分析
---

组件模型和企业级服务是开发中两个重要的方面。前者通过一个轻量级的容器来实现，后者是一个独立的框架，在Spring中提供了允许开发者通过编程方式或者声明方式轻松访问各种企业级服务的能力（AOP），这些服务与轻量级容器紧密结合在一起，但又不是密不可分。这是Spring最大的优势，强大又灵活，这是它之所以能够流行的原因。

在Spring中，组件可以是细粒度的`POJO`也可以是粗粒度的服务`Service`，为了描述方便，统一称为`Bean`。`Bean`是Spring世界中的主要公民，其生命周期包含了：定义、解析注册、实例化、使用、销毁。整个生命周期都在IoC容器的控制范围内，以汽车的制造作为例子：定义相当于是绘制机车制造的图纸，这些图纸需要满足一定的规范和限制约束；解析注册相当于将图纸递交给工厂，工厂根据图纸中的描述，将生产过程分解成为车间能够识别的生产步骤；实例化，在工厂内部，根据上一步得到的生产步骤生产汽车，包括了车身的构建、零配件的安装等；使用，很容易理解；销毁则是当满足一定的条件时，比如使用汽车使用完毕或者工厂倒闭，将已经实例化的汽车对象销毁的过程。

如果把所有应用构建基于Bean，我们就最大限度地提高了自己从应用代码中分离出配置数据的能力。我们也可以保证应用构建能够以一种一致方式被配置，无论配置数据被保存在什么地方。即使我们不知道一个应用在运行时的类（与它所实现的那些接口相比），我们仍知道如何配置它，只要它是一个Java组件。

轻量级容器的主要目的是：在使用`Bean`的过程中，避免和消除众多自制的工厂和Singleton，使用一种统一的方式去装配所有应用对象。借助放射和依赖注入，被`Bean`工厂管理的组件不需要知道Spring的存在。本文会结合源代码和设计方法，详细描述Spring的核心容器设计。

## 容器接口设计

首先是整体IoC容器的接口结构

![beanfactory](/assets/img/spring/beanfactor-1535335071-7572.png)

从上图可以看出，IoC容器的基础类型是`BeanFactory`，并以它为基础，根据不同的场景，拓展出不同的接口实现，

1. BeanFactory：提供最基础和核心的IoC功能`getBean`，也就是根据Bean的名字，从IoC容器中获取实例，这是作为`Factory`模式最基本的功能
2. HierarchicalBeanFactory：BeanFactory可以具有层级，每一个BeanFactory都可以为其定义一个parent级别的BeanFactory，这样整个IoC容器，构成一个继承的树形结构，利用这一点可以实现模块化
3. ListableBeanFactory：提供了根据参数枚举IoC内部Bean的功能
4. ConfigurableBeanFactory：为IoC容器引入配置功能，例如`setParentBeanFactory`设置双亲，`setBeanClassLoader`设置IoC的类加载器，`addBeanPostProcessor`提供后置的等功能
5. AutowireCapableBeanFactory：提供自动注入的增强定义，其中`createBean`定义了根据类的类型定义，完整初始化一个实例的接口方法
6. ApplicationContext：在BeanFactory的基础上，拓展了`MessageSource`支持国际化；`ResourceLoader`可以从不同的地方访问资源；`ApplicationEventPublisher`支持内部的"发布-订阅"模式的事件支持，这些拓展都增强了BeanFactory作为应用上下文提供全面服务的能力
7. ConfigurableApplicationContext：为ApplicationContext，提供了配置功能，其中`refresh`重新加载配置文件并重刷新当前的ApplicationContext，是ApplicationContext这条线中最重要的方法之一
8. WebApplicationContext：提供针对Web应用的配置，其中`getServletContext`用于获取在ApplicationContext启动时候，绑定的`ServletContext`


这边主要分成三类：

1. BeanFactory -> HierarchicalBeanFactory -> ConfigurableBeanFactory 这条线，提供了最基础的IoC容器功能
2. BeanFactory -> ListableBeanFactory -> ApplicationContext -> ConfigurableApplicationContext 这条线，目标是提供在BeanFactory基础上更加高级的容器功能
3. BeanFactory -> ListableBeanFactory -> ApplicationContext -> WebApplicationContext 这条线，是在高级的容器功能上，增加对于Web应用的支持

### Registry

IoC容器除了获取之外，另一个功能就是注册

![registry](/assets/img/spring/registry-1535340170-21172.png)

注册也分成两个部分
1. SingletonBeanRegistry 是Bean的注册，目的是将Bean实例注册容器中，可以在多个使用者之间提供对象共享，一般只针对共享的单例，而对于`Prototype`的Bean，每次请求都会重新生成，因为不需要共享，因此也不需要注册。
2. BeanDefinitionRegistry 是对`BeanDefinition`进行注册，`BeanDefinition`可以理解成为Bean的设计图纸，是IoC容器内对Bean的定义的抽象。一个Bean要被IoC管理，需要统一转换成为`BeanDefinition`。

### BeanFactory

首先看一个基于`DefaultListableBeanFactory`和`XmlBeanDefinitionReader`的例子，我们将使用这个例子展示基础IoC容器功能。

#### 基础准备

第一步，定义XML配置
````xml

    <bean class="com.sinoservices.spring.beans.Man">
        <property name="name" value="SimpleName"/>
        <property name="age" value="10"/>
        <property name="pet" ref="cat" />
    </bean>

    <bean id="cat" class="com.sinoservices.spring.beans.Cat" >
        <property name="name" value="lily" />
    </bean>
````
第二部，编写测试的代码
````java

    ClassPathResource resource = new ClassPathResource("applicationContext.xml");
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    // factory.addBeanPostProcessor(new MyBeanPostProcessor());
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
    reader.loadBeanDefinitions(resource);
    // get Bean from Spring IoC Container
    Person person = factory.getBean(Person.class);
    Pet pet = factory.getBean("cat",Pet.class);
````

> `Person`、`Man`、`Pet`、`Cat`都是简单望文生义的类，这边就不具体说明。

其中`DefaultListableBeanFactory`是上面第一条线的默认实现，也是最常见的`BeanFactory`实现之一

![defaultListable](/assets/img/spring/defaultlis-1535346983-25647.png)

可以看出，`DefaultListableBeanFactory`同时具有`Registry`和`BeanFactory`的身份。`XmlBeanDefinitionReader`则是从XML文件中读取配置，解析成为`BeanDefinition`的工具类。

![BeanDefinition](/assets/img/spring/beandefini-1535347621-18718.png)

`BeanDefinition`的详细定义如上图，比较需要关注的有
1. scope：定义Bean的作用域，默认为单例（Singleton）
2. beanClassName: Bean对应的类名称，有了类名之后，就可以实例化对应的Bean实例
3. propertyValues：Bean实例，对应的属性值，这是IoC容器，进行依赖注入的基础

在这里，我们推测，整个流程应该抽象定义为下面的几个步骤：

1. 在XML声明Bean的定义
2. 从XML中读取定义，转换为BeanDefinition
3. 将BeanDefinition，注册到BeanDefinitionRegistry中
4. 使用Bean名称，从BeanFactory中获取对应的Bean实例
5. BeanFactory根据其中的BeanDefinition实例化Bean，同时完成各种注入和回调操作，返回Bean实例

#### Bean定义到BeanDefinition

`ClassPathResource resource = new ClassPathResource("applicationContext.xml");`

我们的XML文件，名称为`applicationContext.xml`，并存放在`resources`文件夹下，因此使用`ClassPathResource`从中读取，在Spring中`Resource`接口是资源的抽象，在这里使用的是位于classpath的资源，因此选择这个实现类。

````java

    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
    reader.loadBeanDefinitions(resource);

````

定义了XML的BeanDefinition读取器，`XmlBeanDefinitionReader`，为了便于阅读,下面的代码只选择重要的部分，对于校验和异常处理。完整的注释，可以到github上查看

````java

    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        // BeanDefinitionDocumentReader 用于读取DOM，并注册到BeanDefinitionRegistry中
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        // 已经加载的数量
        int countBefore = getRegistry().getBeanDefinitionCount();
        // 执行注册和解析
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        return getRegistry().getBeanDefinitionCount() - countBefore;
    }

    /** DefaultBeanDefinitionDocumentReader **/
    protected void doRegisterBeanDefinitions(Element root) {
    // beans 标签下的所有元素，都会递归调用这个方法
    // 在这一个parent-child的层级结构，用于追踪各个递归层级中的 {@link BeanDefinitionParserDelegate} 代理
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);

        if (this.delegate.isDefaultNamespace(root)) {
            // 使用profile，按照不同的环境，定义需要加载的内容，实现类似Maven或者Spring Boot中Profile的功能
            String profileSpec = root.getAttribute(PROFILE-ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                        profileSpec, BeanDefinitionParserDelegate.MULTI-VALUE-ATTRIBUTE-DELIMITERS);
                if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    return;
                }
            }
        }

        preProcessXml(root);
        // 解析
        parseBeanDefinitions(root, this.delegate);
        postProcessXml(root);

        this.delegate = parent;
    }

    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        // 默认namespace为"http://www.springframework.org/schema/beans"
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        // 默认namespace
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        // 自定义namespace
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        // 自定义namespace
        else {
            delegate.parseCustomElement(root);
        }
    }

    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.nodeNameEquals(ele, IMPORT-ELEMENT)) {
            // 解析import进来的XML
            importBeanDefinitionResource(ele);
        }
        else if (delegate.nodeNameEquals(ele, ALIAS-ELEMENT)) {
            processAliasRegistration(ele);
        }
        else if (delegate.nodeNameEquals(ele, BEAN-ELEMENT)) {
            // 解析 bean 节点
            processBeanDefinition(ele, delegate);
        }
        else if (delegate.nodeNameEquals(ele, NESTED-BEANS-ELEMENT)) {
            // recurse 递归
            doRegisterBeanDefinitions(ele);
        }
    }

    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        // 将Element解析成为DefinitionHolder,这里定义具体的解析逻辑，对比XML的XSD和BeanDefinition，就可以得知其转换过程
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            /** 允许自定义的装饰行为，在BeanDefinition最终注册之前，进行干预 {@link BeanDefinitionDecorator} **/
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // 使用工具类 BeanDefinitionReaderUtils 完成注册
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            // 触发一个已经注册的事件
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }

    public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // Register bean definition under primary name.
        String beanName = definitionHolder.getBeanName();
        /**
        调用{@link BeanDefinitionRegistry } 进行注册，最终回调{@link DefaultListableBeanFactory}
        或者{@link GenericApplicationContext} 实现注册
        **/
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        // 处理别名
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
````

以上的流程可以用下面的时序图说明

![schedule](/assets/img/spring/schedule-1535354686-2653.png)

> 这边需要拓展一下，在上面流程的`parseBeanDefinitions`中的`delegate.parseCustomElement(root);`方法，是用户自定义标签的入口。也是Spring其他模块自定义标签的入口。当XML中的元素，不是使用默认命名空间的时候，就会使用类似`Java SPI`的方式，默认在`"META-INF/spring.handlers"`下查找对应的标签解析器，对自定标签进行解析。

最终，我们将XML文件内的元素一一解析成为`BeanDefinition`，并回调`BeanDefintionRegistry`进行注册，在本例中，就是`DefaultListableBeanFactory`

对应的代码和注释可以参考[xml->beandefinition](https://github.com/fzsens/springframework/commit/fa5440bf74014f560107e91d1ea49e20d50a5d63)

#### Registry注册BeanDefinition

````java

    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {

        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "BeanDefinition must not be null");

        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                ((AbstractBeanDefinition) beanDefinition).validate();
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Validation of bean definition failed", ex);
            }
        }

        BeanDefinition oldBeanDefinition;

    //之前已经注册过
        oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        if (oldBeanDefinition != null) {
           ......// 校验
            // 存放到BeanDefinitionMap中
            this.beanDefinitionMap.put(beanName, beanDefinition);
        }
        else {
            if (hasBeanCreationStarted()) {
                // Bean已经创建过，也就是说执行过getBean操作，进行一个同步操作，避免并发带来的稳定性问题
                synchronized (this.beanDefinitionMap) {
                    this.beanDefinitionMap.put(beanName, beanDefinition);
                    List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
                    updatedDefinitions.addAll(this.beanDefinitionNames);
                    updatedDefinitions.add(beanName);
                    this.beanDefinitionNames = updatedDefinitions;
                    if (this.manualSingletonNames.contains(beanName)) {
                        Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
                        updatedSingletons.remove(beanName);
                        this.manualSingletonNames = updatedSingletons;
                    }
                }
            }
            else {
                // Still in startup registration phase
                this.beanDefinitionMap.put(beanName, beanDefinition);
                this.beanDefinitionNames.add(beanName);
                this.manualSingletonNames.remove(beanName);
            }
            this.frozenBeanDefinitionNames = null;
        }

        if (oldBeanDefinition != null || containsSingleton(beanName)) {
            // 清理缓存，当重新注册一个bean之后，例如之前已经生成的单例对象，需要清除掉
            resetBeanDefinition(beanName);
        }
    }

````

注册动作的最终目的就是在`BeanDefinitionRegistry`中保存beanName和`BeanDefinition`的绑定关系。`DefaultListableBeanFactory`中使用一个`ConcurrentHashMap`来存储，`this.beanDefinitionMap.put(beanName, beanDefinition)`执行这个动作。

#### 获取Bean实例

`DefaultListableBeanFactory`的`getBean`会委托给`AbstractBeanFactory`的`doGetBean`方法

````java

    protected <T> T doGetBean(
            final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
            throws BeansException {

      // beanName，主要处理FactoryBean的&符号
        final String beanName = transformedBeanName(name);
        Object bean;

        // Eagerly check singleton cache for manually registered singletons.
        Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            if (logger.isDebugEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            // 如果bean是 {@link FactoryBean} 还需要进一步处理
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }

        else {
            // Fail if we're already creating this bean instance:
            // We're assumably within a circular reference.
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

      // 尝试从父级 BeanFactory 中获取 Bean
            BeanFactory parentBeanFactory = getParentBeanFactory();
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                if (args != null) {
                    // Delegation to parent with explicit args.
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
            }
            // doGetBean操作是否只是为了完成类型检查，一般都是false
            if (!typeCheckOnly) {
                // 标记Bean已经创建，或者即将被创建，这有助于在重复创建一个类型对象的时候，提前进行缓存优化
                markBeanAsCreated(beanName);
            }

            try {
               // 根据beanName => BeanDefinition
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                checkMergedBeanDefinition(mbd, beanName, args);

         // 保证当前bean依赖的bean会先初始化
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dependsOnBean : dependsOn) {
                        if (isDependent(beanName, dependsOnBean)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
                        }
                        // 创建依赖Bean和被依赖Bean之间的关系，存储在 {@link dependentBeanMap} 和 {@link dependenciesForBeanMap} 中
                        registerDependentBean(dependsOnBean, beanName);
                        // 触发递归
                        getBean(dependsOnBean);
                    }
                }

                // Create bean instance.
                if (mbd.isSingleton()) {
                    // 单例模式，共享一个对象
                    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
            /**
             * 默认的 {@link ObjectFactory} 实现，当单例为 {@link null}的时候，会调用这个方法进行初始化bean
             */
                        @Override
                        public Object getObject() throws BeansException {
                            try {
                                // 创建bean
                                return createBean(beanName, mbd, args);
                            }
                            catch (BeansException ex) {
                                // Explicitly remove instance from singleton cache: It might have been put there
                                // eagerly by the creation process, to allow for circular reference resolution.
                                // Also remove any beans that received a temporary reference to the bean.
                                destroySingleton(beanName);
                                throw ex;
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }

                else if (mbd.isPrototype()) {
                    // 原型模式，每次调用创建一个新的
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        // 单例创建时候的回调
               // 默认是在{@link ThreadLocal} 变量 {@link prototypesCurrentlyInCreation} 中设置当前beanName
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        // 创建后回调
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }

                else {
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
            /**
             * 其他自定义作用域的处理，通过自定义{@link Scope}来实现
             */
                        Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                            @Override
                            public Object getObject() throws BeansException {
                                beforePrototypeCreation(beanName);
                                try {
                                    return createBean(beanName, mbd, args);
                                }
                                finally {
                                    afterPrototypeCreation(beanName);
                                }
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // Check if required type matches the type of the actual bean instance.
        if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
            // 类型检查
            try {
                return getTypeConverter().convertIfNecessary(bean, requiredType);
            }
            catch (TypeMismatchException ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Failed to convert bean '" + name + "' to required type [" +
                            ClassUtils.getQualifiedName(requiredType) + "]", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }

    protected Object  getSingleton(String beanName, boolean allowEarlyReference) {
    /**
     * 使用 double-check lock 处理并发，从 {@link singletonObjects} 中获取单例
     */
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    /**
     * 如果singletonObjects，并且对应的单例bean实例正在创建中，尝试从{@link earlySingletonObjects }获取
     * 这样处理，主要为了解决循环依赖的问题
     * 1. A 依赖于 B ，B 依赖 C，C 依赖 A 形成一个循环，正常在创建的时候，会导致一个死循环
     * 2. 解决的思路，先创建实例，然后提前暴露，存储在 {@link earlySingletonObjects } 中
     * 3. 当A->B->C，实例化 C 的时候，就可以顺利获取 A 的引用，完成实例化C
     * 4. 由于实例之间的引用关系，最终可以顺利实现初始化和关系闭环
     */
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        // 使用{@link singletonFactory} 中对应bean的工厂类创建实例
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return (singletonObject != NULL-OBJECT ? singletonObject : null);
    }

````

`doGetBean`方法很长，其核心在于定义一个创建Bean实例的模板

1. 通过`getSingleton`来获取单例，如果存在缓存，则直接返回，否则调用`ObjectFactory`的`getObject`方法，从上面的源代码中可以看到，最终调用的是`createBean`
2. 当第一次调用时`getSingleton`，`ObjectFactory`也为空，因此返回值为空，如果当前`BeanFactory`不包含对应的Bean，则尝试逐层调用父级`BeanFactory`
3. 先将依赖Bean实例化
4. 再次调用`getSingleton`，这次会创建`ObjectFactory`匿名内部类，具体的创建方法为`createBean(beanName, mbd, args)`，`createBean`是一个模板方法，具体实现在`AbstractAutowireCapableBeanFactory`中

````java

    protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        if (logger.isDebugEnabled()) {
            logger.debug("Creating instance of bean '" + beanName + "'");
        }
        RootBeanDefinition mbdToUse = mbd;

        // Make sure bean class is actually resolved at this point, and
        // clone the bean definition in case of a dynamically resolved Class
        // which cannot be stored in the shared merged bean definition.
        Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }

        // Prepare method overrides.
        try {
            mbdToUse.prepareMethodOverrides();
        } catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                    beanName, "Validation of method overrides failed", ex);
        }

        try {
            // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
            // 使用BeanPostProcessor干预Bean的生命周期，这是初始化阶段
            // 对应的Processor为{@link InstantiationAwareBeanPostProcessor}
            Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
            if (bean != null) {
                return bean;
            }
        } catch (Throwable ex) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                    "BeanPostProcessor before instantiation of bean failed", ex);
        }

        // 默认的初始化
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isDebugEnabled()) {
            logger.debug("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
````

`createBean`定义了一些校验和拓展的方法，实际执行创建实例`doCreateBean(beanName, mbdToUse, args)`方法

````java

    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
        // Instantiate the bean.
        // BeanWrapper 用于封装创建出来的Bean，主要用途在自动注入环节
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        }
        if (instanceWrapper == null) {
            /**
             * 创建实例
             */
            instanceWrapper = createBeanInstance(beanName, mbd, args);
        }
        final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
        Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

        // Allow post-processors to modify the merged bean definition.
        synchronized (mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                /**
                 * {@link BeanPostProcessor} 拓展点，{@link MergedBeanDefinitionPostProcessor}
                 */
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                mbd.postProcessed = true;
            }
        }

        // Eagerly cache singletons to be able to resolve circular references
        // even when triggered by lifecycle interfaces like BeanFactoryAware.
        // 提前缓存单例，用于避免循环引用，具体的原因和逻辑参考之前代码的注释
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            if (logger.isDebugEnabled()) {
                logger.debug("Eagerly caching bean '" + beanName +
                        "' to allow for resolving potential circular references");
            }
            addSingletonFactory(beanName, new ObjectFactory<Object>() {
                @Override
                public Object getObject() throws BeansException {
                    return getEarlyBeanReference(beanName, mbd, bean);
                }
            });
        }

        // Initialize the bean instance.
        Object exposedObject = bean;
        try {
            // 封装bean实例，执行具体的依赖注入
            populateBean(beanName, mbd, instanceWrapper);
            if (exposedObject != null) {
                // 对Bean实例，调用Factory的各种回调函数，包括各类Aware注入，以及init-method
                exposedObject = initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable ex) {
            if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
                throw (BeanCreationException) ex;
            } else {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
            }
        }

        if (earlySingletonExposure) {
            /**
             * 用于处理循环引用问题
             */
            Object earlySingletonReference = getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                    String[] dependentBeans = getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                    for (String dependentBean : dependentBeans) {
                        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }
                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName,
                                "Bean with name '" + beanName + "' has been injected into other beans [" +
                                        StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                        "] in its raw version as part of a circular reference, but has eventually been " +
                                        "wrapped. This means that said other beans do not use the final version of the " +
                                        "bean. This is often the result of over-eager type matching - consider using " +
                                        "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }

        // Register bean as disposable.
        // 注册destroy-method的回调
        try {
            registerDisposableBeanIfNecessary(beanName, bean, mbd);
        } catch (BeanDefinitionValidationException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
        }

        return exposedObject;
    }
````

`doCreateBean`中主要对`BeanPostProcessor`拓展点和`init-method`和`destroy-mehod`以及各类`Aware`注入，进行处理。核心的初始化则由`createBeanInstance(beanName, mbd, args)`实现，而组装对象，实现控制反转，则由`populateBean(beanName, mbd, instanceWrapper)`实现。

`createBeanInstance`方法的调用逻辑非常冗长，并不适合写成blog，其本质为在上下文中查找构造`Bean`所需要的参数（如果有），然后使用反射或者动态代理，生成实例。具体的代码和注释参考[createBeanInstance](https://github.com/fzsens/springframework/commit/8977b042d9c26eaa6573900d880eff0e271ce26c)

`Bean`实例化后，接着处理`Bean`对象之间的依赖关系，这些依赖关系，在`XML->BeanDefinition`阶段，就已经解析到`BeanDefinition`了。`populateBean`主要分成两个步骤

1. `autowireByName`或者`autowireByType`，从`BeanDefinition`中解析到目标`Bean`所依赖的属性和值，如果是`ByName`，直接从`BeanFactory`中`getBean`获取，如果是`ByType`则，从上下文中，查找符合对应类型的实例。最后构造称为`PropertyValues`进入下一个步骤
2. 根据上一个步骤得到的`PropertyValues`，对实例化的`Bean`进行属性设置，完成依赖注入

同样以为这块的代码比较冗长，可以参考[populateBean](https://github.com/fzsens/springframework/commit/238fe2c4db7ccdbff870ad94f4a942e84cfea2af)

整体的时序图可以概况如下

![](/assets/img/spring/1535386737-1874883956.png)

![](/assets/img/spring/1535386777-1995460864.png)


### ApplicationContext

`ApplicationContext`系列容器在`DefaultBeanFactory`的基础上提供了更多的功能，相对也更加重量级一点，通过他的初始化，就可以一窥究竟。通过一些列的调用后，会来到`AbstractApplicationContext`的`refresh`方法，这定义了整个`ApplicationContext`系列容器的初始化骨架。

````java

    // Prepare this context for refreshing.
    // 前置的一些资源准备
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    // 获取内部的BeanFactory，ApplicationContext系列，对于Bean的操作，都委托给内部的BeanFactory
    // 采用 Decorator 设计模式，初始化也是围绕内部的 beanFactory 来完成
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    // 为BeanFactory，配置一些前置资源，例如类加载器、环境变量等等
    prepareBeanFactory(beanFactory);

    // Allows post-processing of the bean factory in context subclasses.
    // 允许不同的实现类，添加一些 post-process，
    // 例如 {@link GenericWebApplicationContext} 添加 ServletContextAwareProcessor，用于注入ServletContext，
    // 环境变量，以及关于Web端的特性作用域
    postProcessBeanFactory(beanFactory);

    // Invoke factory processors registered as beans in the context.
    // 调用注册到BeanFactory中的 factory processors,Spring 将大部分对象都堪称Bean注册到BeanFactory中
    invokeBeanFactoryPostProcessors(beanFactory);

    // Register bean processors that intercept bean creation.
    // 注册 bean processors 到beanFactory，如果使用{@link DefaultListableBeanFactory} 注册需要手工完成
    registerBeanPostProcessors(beanFactory);

    // Initialize message source for this context.
    // 初始化 {@link MessageSource}
    initMessageSource();

    // Initialize event multicaster for this context.
    // 初始化事件广播 {@link ApplicationEventMulticaster}
    initApplicationEventMulticaster();

    /**
     *
     * 以上两个bean都是注册在BeanFactory中的普通对象，如果用户没有默认注册，则会提供一个默认的实现
     *
     * **/

    // Initialize other special beans in specific context subclasses.
    /**
     * template 方法，提供给具体实现类做回调使用
     */
    onRefresh();

    // Check for listener beans and register them.
    /**
     * 为事件广播，增加监听器，手工添加或者，BeanFactory中的 {@linnk ApplicationListener} 都会加入到监听队列
     * 当调用 {@link #publishEvent(Object)} 时候，可以实现事件监听机制
     **/
    registerListeners();

    // Instantiate all remaining (non-lazy-init) singletons.
    /**
     * 主要的功能是实例化BeanFactory中的Bean，
     * 通过 {@link org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons()}
     * 实现
     */
    finishBeanFactoryInitialization(beanFactory);

    // Last step: publish corresponding event.
    // 最后一步，发布事件 {@link ContextRefreshedEvent} 表示 {@link #refresh}步骤完成
    // 同时激活其他例如生命周期监听器等附加的管理工具
    finishRefresh();

````

可以看出，`ApplicationContext`增加的功能，也基本是围绕着核心的IoC容器展开，一些都是`Bean`，包括生命周期中的各个回调和事件发布器，这样的做法，使得Spring的配置和拓展高度一致，也更加易于使用。对应这块功能的注释可以查看 [applicationContext](https://github.com/fzsens/springframework/commit/3749f574780ee78631f5a603f17b0d7904042fa3) 。

### 总结

最后，在整个代码分析中，有两个设计贯穿其中。

一个是允许在底层`BeanFactory`读取`BeanDefinition`以后进行后处理`post-process`机制。

例如只要实现`BeanFactoryPostProcessor`接口，在引用上下文启动的时候，它们会被首先调用，因此可以动态改变任何其他的bean的声明，一个典型的例子是 `PropertyOverrideConfigurer` 读取属性文件中的内容，并覆盖Bean属性的值。另一种类似的机制是bean实例的后处理，通过`BeanPostProcessor`接口提供，`Bean`的后处理可以用于动态创建AOP代理，切入点可以在当前的应用上下文声明和普通的`Bean`一样，也可以通过注解来配置。正如`BeanFactoryPostProcessor`机制所展示的，Spring将Ioc容器的优势发挥到了机制：就连框架特有的类，bean后处理器，web控制器、AOP拦截器和切入点，也都可以在IOC模型中定义。因为引用代码和框架本身的配置都采用同一种机制，所以Spring的配置具有很高的一致性。

另一个是`BeanWrapper`机制，在IoC容器这种应用场景下，最直接使用对Bean实例设值的就是使用反射，但是直接使用放射，需要按照方法名来触发发射方法，同时这意味着Bean需要处理来自核心反射包的异常，所以异常处理会变得很笨拙，因此引入一个更高级别的抽象`BeanWrapper` 和默认的实现 `BeanWrapperImpl`

![beanwrapper](/assets/img/spring/beanwrappe-1535384908-520068175.png)

其中最终设置实例的属性值的方法位于`AbstractNestablePropertyAccessor`的`setPropertyValues`方法，并将所有的反射方法异常都封装未`BeansException`，这是一个`RuntimeException`，不强制用户捕获。显然这种设计是正确的，因为这个阶段发生的异常，一般直接影响到了`Bean`的初始化，这种致命的错误，通常程序无法自动恢复，强制用户捕获和处理这些异常，只会增加代码的复杂度。

到这边，整个依赖注入的部分，其实IoC功能的核心部分，并不复杂，但是经过多年来的发展，Spring关于这块代码的设计和实现都变得很复杂。特别是作为基础性的框架，用户的需求千差万别，需要在各个地方预留拓展接口，Spring在这一块的设计尤其紧密和复杂，大量的拓展接口定义很容易让人迷失在复杂的实现和继承关系中，所幸Spring的代码质量把控依然非常好，只要能够把握整体的脉络，还是可以理解整个IoC容器的设计思路的。

文章的时序图来自 [Spring 框架的设计理念与设计模式分析]: https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/ 


---
layout: post
title: Spring Rest接口注入
date: 2017-06-12
categories: spring
tags: [spring]
description: Spring动态生成Rest接口代理类，并注入到Spring上下文中
---

最近后台系统逐步从Dubbo的RPC风格转换为HTTP REST风格调用，考虑到服务治理和安全域管理的统一，没有直接使用Dubbox作为升级方案，而是使用在服务提供者和服务消费者中间增加网关代理，并逐步将原来使用Dubbo迁移到新的网关(NGINX) + REST(Resteasy)模式上。改造的一部分难点在于减少已有接口的调整，原来大量的接口编码方式为:

````java

@Autowired
private XxxService service; 

````

在使用dubbo进行开发的时候，dubbo会在调用端生成代理类并自动添加到Spring上下文中，调用的时候，使用面向接口的方式，由Spring自动注入代理类。当迁移到Resteasy上时，也可以生成类似的代理类，但是却只能手工来编写下面的代码。

````java

ResteasyClient client = new ResteasyClientBuilder().build();
ResteasyWebTarget target = client.target("url");
XxxService service = target.proxy(XxxService.class);

````
显然相比原来面向接口+自动注入的方式，这种调用方式比较笨拙。好在可以利用Spring强大的拓展功能很方便地解决这个问题，思路如下：

1. 面向接口编程，保留原来RPC时代，定义API接口的好习惯，只是将接口风格调整为RESTful风格
2. 问题就转换为如何在Spring中根据接口动态生成代理类的类并注入到Spring中
3. 代理类的生成，可以直接使用`ResteasyClient`完成，因此难点在于如何注入生成的Proxy
4. Spring实例化Bean并注入的过程大体可以概括为XML/Annotation -> BeanDefinition -> Instance -> Injection的过程，对于不同的Rest service，主要在Spring中定义对应接口类型的BeanDefiniton，并注入到Srping Context中，Spring就可以帮助我们实现是实例化和注入的过程

因此核心的问题就转换为如何Rest Service接口，生成BeanDefinition并注入到Spring的上下文中。

首先我们定义`RestService`注解，用于标记那些接口需要进行代理

````java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface RestService {
    /**
     * @return rest service path
     */
    String path() default "";
}

````

`@RestService`标记在Rest Service接口上，path即为Rest 服务要调用的url路径

其次，我们需要获取这些Rest Service并转换为`BeanDefinition`,既然使用了注解理所当然地会选择使用自动扫描的方式，Spring提供了`ClassPathBeanDefinitionScanner`和`BeanDefinitionRegistryPostProcessor`用于从ClassPath中获取BeanDefinition，并注册到Spring Context。因此扫描，构建，注册，这三个步骤实际上是一次性完成的，其中`BeanDefinitionRegistryPostProcessor`提供`BeanDefinitionRegistry`用于注册，`ClassPathBeanDefinitionScanner`扫描classpath，获取符合条件的Resouce，并转换为BeanDefinition，注入到`BeanDefinitionRegistry`中。

显然Spring并不知道我们要怎么定义BeanDefinition，因此我们对这两个类分别进行拓展。

````

class JaxRsClientClassPathScanner extends ClassPathBeanDefinitionScanner {

    final private Logger log = LoggerFactory.getLogger(JaxRsClientClassPathScanner.class);

    /**
     * The service url.
     */
    private String serviceUrl;


    /**
     * Creates new instance of {@link JaxRsClientClassPathScanner} class.
     *
     * @param registry the bean definition registry
     */
    public JaxRsClientClassPathScanner(BeanDefinitionRegistry registry) {
        super(registry, false);
        registerFilters();
    }

    /**
     * Sets the service url.
     *
     * @param serviceUrl service url
     */
    public void setServiceUrl(String serviceUrl) {
        this.serviceUrl = serviceUrl;
    }

    /**
     * 自动扫描带有RestService的接口
     */
    protected void registerFilters() {
        addIncludeFilter(new AnnotationTypeFilter(RestService.class));
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
        return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
    }

    /**
     * 对basePackages进行扫描
     */
    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {

        // 获得需要扫描的package下面的 BeanDefinition
        final Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

        if (!beanDefinitions.isEmpty()) {
            processBeanDefinitions(beanDefinitions);
        }

        return beanDefinitions;
    }

    /**
     * 为db统一增加属性配置，并注册FactoryBean
     *
     * @param beanDefinitions the bean definitions
     */
    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {

        for (BeanDefinitionHolder beanDefinition : beanDefinitions) {
            GenericBeanDefinition definition = (GenericBeanDefinition) beanDefinition.getBeanDefinition();

            final String serviceClassName = definition.getBeanClassName();

            /**
             * 修改BeanDefinition
             * 相当于
             *
             * <bean class = "JaxRsClientProxyFactoryBean" >
             *     <property name="serviceClass"></property>
             *     <property name="serviceUrl"></property>
             *     <property name="serviceUrlProvider"></property>
             * </bean>
             */
            log.debug("adding rest service {}, url {}", serviceClassName, serviceUrl);
            definition.setBeanClass(JaxRsClientProxyFactoryBean.class);
            definition.getPropertyValues().add("serviceClass", serviceClassName);
            definition.getPropertyValues().add("serviceUrl", serviceUrl);
        }
    }
}

````

在扫描阶段，我们只带有`@RestService`注解的接口，扫描获取到的BeanDefiniton，在`processBeanDefinitions`中进行进一步的加工，在这里，我们使用FactoryBean来实现，稍后会介绍这个FactoryBean，并设置serviceClassName和serviceUrl，serviceUrl也是对应Rest 服务的URL。

````java
public class JaxRsClientScannerConfigurer implements BeanDefinitionRegistryPostProcessor{

    private String basePackages;

    private String serviceUrl;

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 在BeanDefinitionRegistry初始化并且其他的BeanDefinition已经加载之后调用，，一般用于新增bean definition
        final JaxRsClientClassPathScanner scanner = new JaxRsClientClassPathScanner(registry);
        scanner.setServiceUrl(serviceUrl);
        scanner.setServiceUrlProvider(serviceUrlProvider);
        scanner.scan(basePackages);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 所有的bean definition 已经被加载，但是尚未被实例化， 一般用于为bean definition增加properties
    }
}
````

在`BeanDefinitionRegistry`完成默认的BeanDefinition的定义之后，使用Sacnner注入自定义的BeanDefinition。

最后利用Spring的`FactoryBean`机制，根据serviceClassName和serviceUrl来生成具体的代理类。

````java

@Override
public <T> T createClientProxy(Class<T> serviceClass, String serviceUrl) {

    // creates the client builder instance
    final ClientBuilder builder = clientBuilder();

    // configures the builder
    configure(builder);

    // creates the proxy instance
    return proxy(serviceClass, serviceUrl, builder.build());
}

private <T> T proxy(Class<T> serviceClass, String serviceUrl, Client client) {

    final WebTarget target = client.target(serviceUrl);
    return ((ResteasyWebTarget) target).proxy(serviceClass);
}

````

在配置文件中创建

````xml
<!--配置自动扫描-->
<bean class="com.qiongsong.resteasy.spring.support.JaxRsClientScannerConfigurer" >
    <property name="basePackages" value="com.qiongsong"/>
    <property name="serviceUrl" value="http://localhost:8080/"/>
</bean>

<!--代理生成工厂-->
<bean class="com.qiongsong.resteasy.spring.resteasy.RestEasyClientProxyFactory">
</bean>
````
即可实现对于Rest Service的自动注入
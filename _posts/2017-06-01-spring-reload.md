---
layout: post
title: Spring 拓展
date: 2017-06-01
categories: spring
tags: [spring]
description: Spring 拓展默认参数和动态刷新功能
---

### 拓展Spring的属性配置功能

在工作中广泛地使用Spring framework，相比之前零散的配置文件管理和内部组件关系管理，Spring framework确实有大幅度的改进，但是在默认值和值动态更新两个方面，Spring并没有提供默认的支持，好在Spring本身提供了非常强大的拓展功能，让我们在不修改Spring框架的前提下，能够拓展支持这两项功能。

#### 默认值

使用Spring的`PropertyPlaceholderConfigurer`，将配置文件切分成XML和Properties，XML中有一些参数可以被用户进行调整（比如缓存的多少，监听的端口等），当我们发布一个新的软件版本的时候，引入了一些新的参数，客户不应该手工地合并修改已经定义的配置文件，而应该是XML中包含新的配置参数合理的默认值。

在默认的Spring中，通过`PropertyPlaceholderConfigurer`有两种途径可以实现，第一种在XML写入默认的配置参数；第二种添加第二个配置文件，并将配置文件包括在发布的jar包中。

以上的两种方法都有一些麻烦，实际上得益于`PropertyPlaceholderConfigurer`良好的可拓展性，可以通过下面的方法来实现默认值的设置，并兼容目前的模式。

> `PropertyPlaceholderConfigurer`的可拓展性来源于它是`BeanFactoryPostProcessor`的一个实现类，简单介绍一下`BeanFactoryPostProcessor`的运行原理，Spring容器在加载Bean定义之后，进行Bean的实例化(创建Bean的Instance)之前，会获取容器中所有的`BeanFactoryPostProcessor`，并依次调用，因此我们可以在这边对Bean的定义进行自定义调整，而`PropertyPlaceholderConfigurer`就是一个典型的，在这个阶段对`BeanDefinition`的元数据，也就是，我们在定义Bean的时候写的`Property`参数。

`PropertyPlaceholderConfigurer`进行property的placeholder解析的方法为`resolvePlaceholder`

````java

	protected String resolvePlaceholder(String placeholder, Properties props, int systemPropertiesMode) {
		String propVal = null;
		if (systemPropertiesMode == SYSTEM_PROPERTIES_MODE_OVERRIDE) {
			propVal = resolveSystemProperty(placeholder);
		}
		if (propVal == null) {
			propVal = resolvePlaceholder(placeholder, props);
		}
		if (propVal == null && systemPropertiesMode == SYSTEM_PROPERTIES_MODE_FALLBACK) {
			propVal = resolveSystemProperty(placeholder);
		}
		return propVal;
	}

````

因此，我们可以使用Override这个方法，加入默认值选项，即可实现我们的目的。

````java

	public class DefaultPropertyPlaceholderConfigurer extends PropertyPlaceholderConfigurer {
	
		public static final String DEFAULT_DEFAULT_SEPARATOR = "=";
	
		private String defaultSeparator = DEFAULT_DEFAULT_SEPARATOR;
	
		/**
		 * 设置默认参数分隔符号，默认为"="
		 *
		 * @see #DEFAULT_DEFAULT_SEPARATOR
		 */
		public void setDefaultSeparator(String defaultSeparator) {
			this.defaultSeparator = defaultSeparator;
		}
	
		/**
		 * 覆盖placeholder的替换方法
		 */
		@Override
		protected String resolvePlaceholder(String placeholderWithDefault, Properties props, int systemPropertiesMode) {
			String placeholder = getPlaceholder(placeholderWithDefault);
			String resolved = super.resolvePlaceholder(placeholder, props, systemPropertiesMode);
			if (resolved == null)
				resolved = getDefault(placeholderWithDefault);
			return resolved;
		}
	
		/**
		 *
		 * 从配置的placeholder中，获取真正的替换符
		 *
		 * @see #setPlaceholderPrefix
		 * @see #setDefaultSeparator
		 * @see #setPlaceholderSuffix
		 */
		protected String getPlaceholder(String placeholderWithDefault) {
			int separatorIdx = placeholderWithDefault.indexOf(defaultSeparator);
			if (separatorIdx == -1)
				return placeholderWithDefault;
			else
				return placeholderWithDefault.substring(0, separatorIdx);
		}
	
		/**
		 * 从配置的placeholder中，获取默认值
		 *
		 * @return the default value, or null if none is given.
		 * @see #setPlaceholderPrefix
		 * @see #setDefaultSeparator
		 * @see #setPlaceholderSuffix
		 */
		protected String getDefault(String placeholderWithDefault) {
			int separatorIdx = placeholderWithDefault.indexOf(defaultSeparator);
			if (separatorIdx == -1)
				return null;
			else
				return placeholderWithDefault.substring(separatorIdx + 1);
		}
	}

````

在使用过程中，可以直接使用`DefaultPropertyPlaceholderConfigurer`替换原来的`PropertyPlaceholderConfigurer`


````xml

  <bean id="configproperties"
        class="net.wuenschenswert.spring.ReloadablePropertiesFactoryBean">
    <property name="location" value="file:src/test/resources/spring/example/config.properties"/>
  </bean>

  <bean id="mybean" class="net.wuenschenswert.spring.test.MyBean">
    <property name="cachesize" value="#{cache.size=100}"/>
  </bean>
````

`DefaultPropertyPlaceholderConfigurer`自定义了`PlaceHolder`获取值的过程，如果通过默认的方式在properties中无法读取到对应的参数值的话，会使用 = 号后的值作为默认值。


#### 重新加载

在上面的例子中，我们为配置属性定义了默认值。在系统运行过程中，也很可能会对一些参数进行一些调整。在将Bean定义对象和配置文件分离之后，我们希望在调整配置文件之后，可以对使用了这些配置参数的Bean进行自动更新，这就是我们第二次需求。

Spring 默认的`PropertyPlaceholderConfigurer`设计，刚才也已经提到，通过${}宏语法定义的值，通过key会关联到Properties中的value。如果要在Spring中实现这种功能，需要关闭ApplicationContext，然后重新启动。如何不中断操作，而能够达到效果呢？

实现这个功能的设计目标如下：

* 语法和使用方式接近标准的Spring
* 需要支持不做动态加载的placeholder
* 不适用额外的设置参数线程，配置类应该可以自己控制配置文件何时检测更新
* 面向接口，可测试
* 只需要对singleton bean做设置，对于prototype类型的，每次新建实例的时候，自然会读取到新的配置值。而对于旧的值进行修改没有太多的实际意义。


最终实现的配置效果如下

````xml
<bean id="configproperties" class="net.wuenschenswert.spring.ReloadablePropertiesFactoryBean">
  <property name="location" value="file:config.properties"/>
</bean>

<bean id="propertyConfigurer"
      class="net.wuenschenswert.spring.ReloadingPropertyPlaceholderConfigurer">
  <property name="properties" ref="configproperties"/>
</bean>

<bean id="mybean" class="net.wuenschenswert.spring.example.MyBean">
  <property name="cachesize" value="#{my.cache.size}"/>
</bean>

<!-- regularly reload property files. -->
<bean id="timer" class="org.springframework.scheduling.timer.TimerFactoryBean">
   ...(see complete file for details)...
<bean>
````

下面简要分析一下实现方式，

为了动态加载properties，我们最容易想到的是下面这几个元素

1. 一个Factory Bean，当文件系统发生变化的时候，用于可以执行reload操作。
2. 一个带有观察模式的Properties对象，当文件系统发生变化的时候，可以捕获和生成变化事件并发送给对应的监听器，实际的Properties委托给这个类进行代理，这样可以在内部直接使用`Properties`对象进行值管理的时候。
3. 一个可以管理和记录placeholder到对应属性值的转换过程，记录placeholder/properties/spring bean之间的关系
4. 一个监听器在properties发生更新的时候，进行spring bean的更新。
5. 一个定时器，用于定时检查properties文件是否发生变化，并调用4对应的监听器，进行bean的更新

以上这三个类分别定义了可重载的Properties的创建、对象以及管理。

观察者和代理模式的实现，围绕着`ReloadableProperties`，`ReloadablePropertiesListener`，`PropertiesReloadedEvent`和`ReloadablePropertiesBase`实现，这几个类都是很普通的实现，监听器以及对应的处理，`DelegatingProperties`是一个abstract class覆盖了`Properties`类的所有方法并将其委托给自己的`delegate`内部对象进行操作，可以在properties发生变化的时候，动态地替换掉当前的properties.对properties的更新是一次性完成，因此应用不存在不一致的中间状态。

`ReloadablePropertiesFactoryBean`是用于创建`ReloadableProperties`实例的FactoryBean，完成和spring默认的PropertiesFactoryBean一样的工作。同时，当创建一个新的实例的时候，ReloadablePropertiesFactoryBean都检查所有的配置文件是否修改，如果修改过则会进行Properties的更新。

类结构

![](http://ooi50usvb.bkt.clouddn.com/sprinreload.png)

`ReloadablePropertiesBase`中实现添加和触发监听器，以及设置配置文件对象的入口。同时，在`ReloadablePropertiesFactoryBean`中定义了内部类`ReloadablePropertiesImpl`，其继承了`ReloadablePropertiesBase`，并实现了了`ReconfigurableBean`接口。FactoryBean创建Bean的时候，实际对象未`ReloadablePropertiesImpl`，当`ReloadablePropertiesImpl`进行`reloadConfiguration`的时候，则会调用FactoryBean的`reload`方法，从而触发整个配置文件更新的过程。

ReloadingPropertyPlaceholderConfigurer是IReloadablePropertiesListener的实现类，同时也继承了PropertyPlaceholderConfigurer，它能够追踪placeholder的使用情况，当properties重新加载的时候，所有用到这些properties的bean会被发现，并重新刷新如bean中。在这个方法中使用了BeanFactoryPostProcessor，自定义了对BeanDefinition的解析过程，并在其中建立对整个BeanDefinition中的placeholder和properties的追踪链路。

`ReloadingPropertyPlaceholderConfigurer`作为`BeanFactoryPostProcessor`，在进行placeHolder进行`parseStringValue`调用的时候，完成对placeHolder和Spring Bean的绑定

````java
	/**
	 * BeanDefinition解析获取值
	 */
	protected String parseStringValue(String strVal, Properties props, Set visitedPlaceholders)
			throws BeanDefinitionStoreException {

		//对应本次解析的Spring Bean
		DynamicProperty dynamic = null;

		// replace reloading prefix and suffix by "normal" prefix and suffix.
		// remember all the "dynamic" placeholders encountered.
		StringBuffer buf = new StringBuffer(strVal);
		int startIndex = strVal.indexOf(this.reloadingPlaceholderPrefix);
		while (startIndex != -1) {
			int endIndex = buf.toString().indexOf(this.reloadingPlaceholderSuffix,
					startIndex + this.reloadingPlaceholderPrefix.length());
			if (endIndex != -1) {
				if (currentBeanName != null && currentPropertyName != null) {
					String placeholder = buf.substring(startIndex + this.placeholderPrefix.length(), endIndex);
					placeholder = getPlaceholder(placeholder);
					if (dynamic == null)
						dynamic = getDynamic(currentBeanName, currentPropertyName, strVal);
                    // 添加以来关系，内部通过HashMap placeholderToDynamics 维护关系
					addDependency(dynamic, placeholder);
				} else {
					logger.warn("dynamic property outside bean property value - ignored: " + strVal);
				}
				buf.replace(endIndex, endIndex + this.reloadingPlaceholderSuffix.length(), placeholderSuffix);
				buf.replace(startIndex, startIndex + this.reloadingPlaceholderPrefix.length(), placeholderPrefix);
				startIndex = endIndex - this.reloadingPlaceholderPrefix.length() + this.placeholderPrefix.length()
						+ this.placeholderSuffix.length();
				startIndex = strVal.indexOf(this.reloadingPlaceholderPrefix, startIndex);
			} else
				startIndex = -1;
		}
		// then, business as usual. no recursive reloading placeholders please.
		return super.parseStringValue(buf.toString(), props, visitedPlaceholders);
	}
````

而本身`ReloadingPropertyPlaceholderConfigurer`作为一个`ReloadablePropertiesListener`的实现类，当`ReloadablePropertiesBase`进行properties设置的时候，利用观察者模式，会通知`ReloadingPropertyPlaceholderConfigurer`，并调用其`propertiesReloaded`方法。

````java
	/**
	 * 监听时间出发重新加载
	 */
	public void propertiesReloaded(PropertiesReloadedEvent event) {
		Properties oldProperties = lastMergedProperties;
		try {
			Properties newProperties = mergeProperties();
            //确定发生变化的Properties
			Set<String> placeholders = placeholderToDynamics.keySet();
			Set<DynamicProperty> allDynamics = new HashSet<DynamicProperty>();
			for (String placeholder : placeholders) {
				String newValue = newProperties.getProperty(placeholder);
				String oldValue = oldProperties.getProperty(placeholder);
				if (newValue != null && !newValue.equals(oldValue) || newValue == null && oldValue != null) {
					if (logger.isInfoEnabled())
						logger.info("Property changed detected: " + placeholder
								+ (newValue != null ? "=" + newValue : " removed"));
					List<DynamicProperty> affectedDynamics = placeholderToDynamics.get(placeholder);
					allDynamics.addAll(affectedDynamics);
				}
			}
			// sort affected bean properties by bean name and say hello.
			Map<String, List<DynamicProperty>> dynamicsByBeanName = new HashMap<String, List<DynamicProperty>>();
			Map<String, Object> beanByBeanName = new HashMap<String, Object>();
			for (DynamicProperty dynamic : allDynamics) {
				String beanName = dynamic.getBeanName();
				List<DynamicProperty> l = dynamicsByBeanName.get(beanName);
				if (l == null) {
					dynamicsByBeanName.put(beanName, (l = new ArrayList<DynamicProperty>()));
					Object bean = null;
					try {
						// 获取收到影响的bean
						bean = applicationContext.getBean(beanName);
						beanByBeanName.put(beanName, bean);
					} catch (BeansException e) {
						// keep dynamicsByBeanName list, warn only once.
						logger.error("Error obtaining bean " + beanName, e);
					}
					try {
                        // 前置切面
						if (bean instanceof ReconfigurationAware)
							((ReconfigurationAware) bean).beforeReconfiguration(); // hello!
					} catch (Exception e) {
						logger.error("Error calling beforeReconfiguration on " + beanName, e);
					}
				}
				l.add(dynamic);
			}
			// for all affected beans...
			Collection<String> beanNames = dynamicsByBeanName.keySet();
			for (String beanName : beanNames) {
				Object bean = beanByBeanName.get(beanName);
				if (bean == null) // problems obtaining bean, earlier
					continue;
                // Spring BeanWrapper
				BeanWrapper beanWrapper = new BeanWrapperImpl(bean);
				// for all affected properties...
				List<DynamicProperty> dynamics = dynamicsByBeanName.get(beanName);
				for (DynamicProperty dynamic : dynamics) {
					String propertyName = dynamic.getPropertyName();
					String unparsedValue = dynamic.getUnparsedValue();

					// obtain an updated value, including dependencies
					String newValue;
					removeDynamic(dynamic);
					currentBeanName = beanName;
					currentPropertyName = propertyName;
					try {
						newValue = parseStringValue(unparsedValue, newProperties, new HashSet());
					} finally {
						currentBeanName = null;
						currentPropertyName = null;
					}
					if (logger.isInfoEnabled())
						logger.info("Updating property " + beanName + "." + propertyName + " to " + newValue);
					try {
                        //赋值
						beanWrapper.setPropertyValue(propertyName, newValue);
					} catch (BeansException e) {
						logger.error("Error setting property " + beanName + "." + propertyName + " to " + newValue, e);
					}
				}
			}
            // 后置切面
			for (String beanName : beanNames) {
				Object bean = beanByBeanName.get(beanName);
				try {
					if (bean instanceof ReconfigurationAware)
						((ReconfigurationAware) bean).afterReconfiguration();
				} catch (Exception e) {
					logger.error("Error calling afterReconfiguration on " + beanName, e);
				}
			}
		} catch (IOException e) {
			logger.error("Error trying to reload properties: " + e.getMessage(), e);
		}
	}
````

整个流程非常简单，接受到变更事件之后，首先确定发生变化的Properties，然后获取之前在解析阶段绑定的DynamicProperty对象，从Spring上下文中获取对应的Bean实例，并讲属性设置未新的配置参数。

整个动态刷新的过程到这边就完成了，Spring在进行整个Bean的管理过程中，充分考虑了拓展的需求，在整个Bean管理生命周期的划分为相互解耦的多个周期，并在各个周期提供便利的拓展手段。而监听者模式，也非常适合作为这类通知类的需求。综合两者，我们完成了spring的bean动态重载功能。

代码可以参考 [SpringPropertiesReloaded](https://github.com/fzsens/SpringPropertiesReloaded)






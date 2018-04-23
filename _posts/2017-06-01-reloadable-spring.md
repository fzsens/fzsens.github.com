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
        class="com.qiongsong.spring.addons.ReloadablePropertiesFactoryBean">
    <property name="location" value="file:src/test/resources/spring/example/config.properties"/>
  </bean>

  <bean id="mybean" class="com.qiongsong.spring.addons.test.MyBean">
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









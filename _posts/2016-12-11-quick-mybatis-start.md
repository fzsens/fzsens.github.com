---
layout: post
title: 快学Mybatis-上手使用
date: 2016-12-11
categories: mybatis
tags: [mybatis]
description: 介绍Mybatis常用使用方法
---

>上一章花费了大量时间描述了Mybatis的产生背景，本章主要参考[Mybatis的官方用户文档](https://www.mybatis.org/mybatis-3/zh/getting-started.html) 简要表述Mybatis的使用方法，并使用一个例子贯穿进行说明和讲解。
>如果想要彻底了解Mybatis的使用和配置方式，都可以通过官方用户文档来获取相关的配置资料，本章只适合作为导读

### 创建SqlSessionFactory

在JDBC中，操作数据库的接口为`Connection`，在Mybatis应用中，这个接口被`SqlSession`封装，而产生`SqlSession`的对象则是`SqlSessionFactory`，从名称就可以看出是一个工厂类。每一个Mybatis应用（单个数据源），一般都有且仅有一个`SqlSessionFactory`实例，并围绕这个实例构建应用的数据库操作层。

因此要使用Mybatis首先要创建一个`SqlSessionFactory`。而`SqlSessionFactory`一般都是使用`SqlSessionFactoryBuilder`创建，`SqlSessionFactoryBuilder`又可以使用XML或者编程方式的`Configuration`创建。

>在Java编程体系中，XML作为一种标记语言被广泛使用作为配置文件的定义，但是实际上在后台都会有对应的`Translator`程序，将XML翻译称为对应的Java对象式的数据结构，才能被系统真正读取，例如Spring中的`Bean`都会被翻译称为`BeanDefinition`实例。

此处主要为了说明Mybatis这几个核心对象之间的关系，因此使用编程方式来进行配置，同时也会给出对应的`XML`配置，但是后者不会做过多的说明。

````java
//1. create datasource
BasicDataSource ds = new BasicDataSource();
ds.setDriverClassName("com.mysql.jdbc.Driver");
ds.setUrl("jdbc:mysql://localhost:3306/test");
ds.setUsername("root");
ds.setPassword("123456");
//2. create transcationManager
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, ds);
Configuration configuration = new Configuration(environment);
SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
//3. create sqlSessionFactory instance
SqlSessionFactory sqlSessionFactory = builder.build(configuration);
````

要创建一个`SqlSessionFactory`，两个核心为：用于获取数据库连接的`DataSource`；决定事务作用域和操作的`TransactionManager`。

对应的XML配置为

````xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
	<environments default="development">
		<environment id="development">
			<transactionManager type="JDBC"/>
			<dataSource type="POOLED">
				<property name="driver" value="${driver}"/>
				<property name="url" value="${url}"/>
				<property name="username" value="${username}"/>
				<property name="password" value="${password}"/>
			</dataSource>
		</environment>
	</environments>
</configuration>
````

可以看到XML的配置基本和编程模式是一一对应的关系。

### 获取SqlSession

获取`sqlSessionFactory`之后，通过

````java
SqlSession sqlSession = sqlSessionFactory.openSession();
````

获取`SqlSession`实例，`SqlSession`是Mybatis的一个核心接口，用于数据库操作和事务的管理，同时也用于获取映射实例`Mapper`实例，对于数据库的直接操作都是围绕着`SqlSession`接口展开。

与`Connection`必须通过`Statement`对象来执行SQL类似，`SqlSession`也不能直接执行SQL语句。Mybatis在`SqlSession`中提供了一系列接口都是通过唯一的`StatementId`（语句ID）和其对应的参数、处理器，从而获取写在XML中的SQL语句并执行。

>如果想要越过`SqlSession`接口的限制，则可以选择通过`SqlSession.getConnection()`直接获取原生的`Connection`对象进行数据库的操作

### 使用SqlSession进行数据库操作

首先让我们先假设一个简单的业务场景：

>`Author`包含三个字段（id、name、blocked）,用于记录作者的基本信息。需要完成这张表的增删改查操作。

首先定义`Author`的数据抽象，很简单的`POJO`

````java
public class Author {
  private long    id;
  private String  name;
  private boolean blocked;
  /*ignore getter setter*/
}
````

其次定义`Author`的`Mapper`接口，用于绑定和标识XML/Annotation中的SQL语句

````java
public interface AuthorMapper {
    public int count();
    public List<Author> selectList();
    public int insert(@Param("name") String name, @Param("blocked") int blocked);
    public int delete(@Param("name") String name);
    public int update(@Param("name") String name, @Param("blocked") int blocked);
}
````

在这边使用最常见的XML配置的方式，因此需要在同一个`package`目录下创建Mybatis `MapperXML`配置

````xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.qiongsong.mybatis.mapper.AuthorMapper">
    <select id="selectList" resultType="com.qiongsong.mybatis.domain.Author">
    	select * from author;
    </select>
    <select id="count" resultType="int">
        select count(1) from author;
    </select>
    <insert id="insert">
        insert into author (name,blocked) values(#{name}, #{blocked})  ;
    </insert>
    <delete id="delete">
        delete from author where name = #{name} ;
    </delete>
    <update id="update">
       update author set blocked=#{blocked} where name = #{name} ;
    </update>
</mapper>
````

在编写XML的时候，有两个点需要注意：`MapperXML`的`namespace`对应到`AuthorMapper`的全限定名称；每一个XML标签都有一个ID，对应到`AuthorMapper`接口的方法名。如果是简单的操作，也可以使用`Annotation`注解的方式将两者合二为一。

````java
public interface AuthorMapper {
    @Select("select count(1) from author;")
    public int count();
    @Select("select * from author;")
    public List<Author> selectList();
    @Insert("insert into author (name,blocked) values(#{name}, #{blocked}) ;")
    public int insert(@Param("name") String name, @Param("blocked") int blocked);
    @Delete("delete from author where name = #{name} ;")
    public int delete(@Param("name") String name);
    @Update("update author set blocked=#{blocked} where name = #{name} ;")
    public int update(@Param("name") String name, @Param("blocked") int blocked);
}
````

到目前为止，定义了操作数据库语句`XML SQL`，以及这些语句的定位标识`Mapper`接口，这两部分构成了Mybatis的操作映射器。为了能对数据库进行操作，还需要将映射器与Mybatis框架建立关联，我们可以通过在`Configuration`定义的时候，进行`Mapper`接口注入来达到绑定的作用。

````java
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, ds);
Configuration configuration = new Configuration(environment);
configuration.addMapper(AuthorMapper.class);
SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
SqlSessionFactory sqlSessionFactory = builder.build(configuration);
````

之后便可以通过SqlSession获取映射器实例`Mapper`接口，来进行数据库操作。

````java
AuthorMapper authorMapper = sqlSession.getMapper(AuthorMapper.class);
int count = authorMapper.count();
int insetedRow = authorMapper.insert("Yuhua", 0);
int updatedRow = authorMapper.update("Yuhua", 1);
int deletedRow = authorMapper.delete("Yuhua");
List<Author> list = authorMapper.selectList();
````

>执行上面的SQL也可以使用类似`List<Author> list = sqlSession.selectList("com.qiongsong.mybatis.mapper.AuthorMapper.selectList");`的方式，后续原理解析会解释为什么可以这样。

### 作用域的控制

要了解作用域，首先要明白上下文`Context`的定义，我们在编写程序或者使用Tomcat或者Spring等软件/框架的时候，都经常听到上下文`Context`这个说法，那要怎么理解程序的上下文和作用域？

在相对论中，时间和空间构成了一个事件发生的准确坐标，于是时间和空间结合在一起形成了时空的概念，在一个时空中发生的事件，无法被另一个时空的观察者感知。对于编程来说，代码执行的时间和数据标识的空间也是不可分割的。只有把指令执行额具体时刻和数据标识映射的具体地址结合起来，才能确定程序在执行瞬间的状态，代码运行时刻与数据构成了上下文状态的概念。在程序指令执行时能访问到的数据集合就构成了当前的上下文。

而数据有效的时间和空间范围就构成了作用域的定义。对于Java而言，有以下几种作用域定义：

- 类级别的作用域：使用`static`修饰的类变量，在整个JVM实例运行的时间内，在装载这个类的`Classloader`范围内都可以访问变量；
- 对象实例级别：普通的成员变量，在初始化实例的时候完成初始化，到实例对象被JVM回收之前，能通过对象实例访问变量；
- 方法级别：在方法内部定义的临时变量，在临时变量，到方法退出前，在方法体内都是临时变量的作用域；
- 块级别：使用{}包裹代码块中定义的变量，只在{}包裹的代码执行时在{}包裹的代码块内生效

前两个作用域，可以被不同的线程同时访问，后两个作用域在同一时刻只能被一个线程访问。

了解作用域和上下文的重要意义在于，编写代码的时候，需要时刻注意当前代码在运行时能访问和读取的数据，以及可能影响的范围。对于资源管理（创建、回收），以及资源使用（并发访问，状态一致性控制）尤为重要。事务的控制也是基于此，只有并发才会带来资源的竞争，只有相同的上下文才存在一致性的问题。

对于Myabtis中几个核心对象的作用域如下

1.	SqlSessionFactoryBuilder：
这个类可以被实例化、使用和丢弃，一旦创建了`SqlSessionFactory`，就不再需要它了。因此 `SqlSessionFactoryBuilder`实例的最佳范围是方法范围（也就是局部方法变量）。你可以重用 `SqlSessionFactoryBuilder`来创建多个`SqlSessionFactory`实例，但是最好还是不要让其一直存在，以保证所有的XML解析资源开放给更重要的事情。

2.	SqlSessionFactory：
`SqlSessionFactory`一旦被创建就应该在应用的运行期间一直存在，没有任何理由对它进行清除或重建。使用`SqlSessionFactory`的最佳实践是在应用运行期间不要重复创建多次，因此 `SqlSessionFactory`的最佳作用域是应用范围。有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。(实践中使用IOC框架的singleton来实现最为普遍，后面说到和Spring的集成会说到)

3.	SqlSession：
每个线程都应该有它自己的`SqlSession`实例。`SqlSession`的实例不是线程安全的，因此是不能被共享的，所以它的最佳的范围是请求或方法范围。也就是说`SqlSession`作为资源必须被单线程持有，因此，`SqlSession`的作用域不能类级别也不能是实例级别，如果是Web应用中，每一个请求都会对应到一个特定的操作线程，在整个请求生命周期中，该线程是请求独占的，因此在`Request`打开一个`SqlSession`，在`Request`返回响应之前关闭。因为`SqlSession`中包含对`Connection`的管理，因此对于`SqlSession`这个关闭操作是很重要的，特别是为了提供系统的整体性能，经常会在数据源层加入连接池技术，如果没有对`Connection`进行妥善的管理，可能会影响整个系统的运行。应该把这个关闭操作放到`finally`块中以确保每次都能执行关闭。下面的示例就是一个确保`SqlSession`关闭的标准模式：

	````java
	SqlSession session = sqlSessionFactory.openSession();
	try {
	  // do work
	} finally {
	  session.close();
	}
	````

4.	映射器实例（Mapper Instances）：
映射器是创建用来绑定映射语句的接口。映射器接口的实例是从 `SqlSession`中获得的。因此从技术层面讲，映射器实例的最大范围是和`SqlSession`相同的，因为它们都是从`SqlSession`里被请求的。尽管如此，映射器实例的最佳范围是方法范围。也就是说，映射器实例应该在调用它们的方法中被请求，用过之后即可废弃。为了保持简单，最好把映射器放在块作用域内

	````java
	SqlSession session = sqlSessionFactory.openSession();
	try {
	  BlogMapper mapper = session.getMapper(BlogMapper.class);
	  // do work
	} finally {
	  session.close();
	}
	````

### 事务控制

数据库事务控制的核心在于保证在有多个相互关联的数据库操作的时候，能够将这个个操作视为一个单元，保证全部成功，或者全部失败，而不存在中间状态。

现代的RDBMS对于事务的管理非常复杂，由锁、log、多版本控制等复杂技术支撑的数据库系统支撑着整个数据库事务体系的运行。在这边我们不深入讨论关于数据库是如何实现事务的管理和控制的，（这个主题可以留待以后进行讨论）。而主要针对Mybatis如何进行事务的控制。

首先，在应用中访问数据库的API是JDBC，而直接连接数据库进行数据操作的接口则是`Connection`，一个标准的数据库更新操作如下

````java
boolean autoCommitDefault = conn.getAutoCommit();
try {
    conn.setAutoCommit(false);
	Statement statement = conn.createStatement();
     statement.execute("insert into author values('Murakami',57)");
	statement.execute("insert into author values('Fizzle',60)");
    /* 1. You execute statements against conn here transactionally */
    conn.commit();
} catch (Throwable e) {
    try { conn.rollback(); } catch (Throwable ignore) {}
    throw e;
} finally {
    try { conn.setAutoCommit(autoCommitDefault); } catch (Throwable ignore) {}
}
````

在这个简单的例子中，需要特别关注的是`autoCommit`、`commit()`、`rollback()`。`autoCommit` 标识`connection`在执行一个SQL之后，是否马上提交，如上面例子中，

1.	如果设置`autoCommit`为`true`，则在执行`statement.execute`方法后，`connection`会立刻发起`commit`，因此每一个语句之间是彼此独立的，并不在同一个事务中。
2.	如果设置`autoCommit`为`false`，设置之后，就可以利用`statement`进行数据库的操作，如果有多个SQL在此处执行，则会被当作一个逻辑执行单元，直到调用`commit()`的时候，一并提交。
3.	`statement`在执行的时候，可能抛出异常，如果设置`autoCommit`为`true`的情况下，抛出异常的语句不会执行，而已经执行的语句则已经生效并且无法回滚；如果设置为`false`，则可以在异常处理中，使用`connection`的`rollback()`来进行回滚操作，则整个逻辑执行单元中的修改操作都会被回滚。

由此可知，JDBC中关于事务操作的核心接口就是`Connection`，而围绕着JDBC之上构建的应用关于事务的操作也围绕着`Connection`以及他的一个属性：`autoCommit`和两个方法：`commit()`，`rollback()`展开。

回到Mybatis中，在创建`SqlSessionFactory`的时候，我们指定了一个类型为`JdbcTransactionFactory`的`TransactionFactory`对象。

````java
TransactionFactory transactionFactory = new JdbcTransactionFactory();//JDBC
````

这个对象的主要作用就是为了管理Mybatis中的事务，在Mybatis中`TransactionFactory`还有另外一个实现`ManagedTransactionFactory`。

````java
TransactionFactory transactionFactory = new ManagedTransactionFactory();//MANAGED
````

使用`MANAGED`的方式，则事务的控制全部交给`DataSource`的`Connection`来进行控制。

使用JDBC的方式，则事务的控制可以在打开`SqlSession`的时候，通过传入`autoCommit`参数进行控制，如果使用`true` 则默认行为等于当前Connection设置为`autoCommit=true`。如果设置为`false`，则等同于`autoCommit=false`。

````java
SqlSession sqlSession = sqlSessionFactory.openSession(true);
````

默认的`SqlSessionFactory.openSession()`获取的`SqlSession`的对象为`autoCommit=false`。在逻辑执行单元执行完成之后，需要调用`sqlSession.commit();`进行提交操作。

### 与Spring集成

在前面的章节中，我们都使用最基本的Mybatis配置，不依赖于其他的框架。很多繁琐的细节，对象的创建和管理都需要我们亲历亲为，而又另外一个框架正是为了解决这个问题而诞生的，相信大家也都非常的熟悉 — Spring Framework。这个章节将介绍如何将之前手工创建和管理Mybatis对象，和Spring框架无缝集成在一起，从而简化Mybatis使用。
>在Spring体系中，有两种主流的方式用于管理配置，JavaConfig以及XML，在这边主要选择XML进行说明。

定义`Configuration`对应的XML配置，`mybatis-config.xml`

````xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="lazyLoadingEnabled" value="false"/>
        <setting name="cacheEnabled" value="true"/>
        <setting name="mapUnderscoreToCamelCase" value="true" />
    </settings>
</configuration>
````

更多的配置添加到Mybatis的核心类创建XML中，是一个标准的Spring `applicationContext.xml`

````xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="configLocation" value="classpath:META-INF/mybatis-config.xml"/>
</bean>
<bean id="sqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg index="0" ref="sqlSessionFactory"/>
</bean>
<bean id="mapperScannerConfigurer" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value=" com.qiongsong.**.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
</bean>
````

配置文件由三个部分组成：`SqlSessionFactory`，`SqlSessionTemplate`，`MapperScannerConfigurer`。其中`SqlSessionFactory`通过数据源和`Configuration`构造，整个应用中只需要创建一个（Spring的Bean默认为单例）。

`SqlSessionTemplate`用于在Spring管理下，创建和管理Mybatis`SqlSession`对象。在应用中也是单例模式。

`MapperScannerConfigurer`用于自动扫描和注入映射器`Mapper`，利用Spring的接口扫描功能，默认情况下b`asePacakge`中所有的接口都会被作为映射器扫描注册。并生成对应的`MapperFactoryBean`代理类，通过Spring的自动注入机制，就可以在整个Spring上下文中使用。

为什么使用`SqlSessionTemplate`而不是默认的`DefaultSqlSession`？在前面的章节中有说到`SqlSession`最好设置为线程独占的模式，而在Mybatis-Spring中却配置为单例？以及`MapperScannerConfigurer`的工作模式，会在后面的章节中说明。

事务的配置，可以使用`@Transactional`注解，或者在XML配置如

````xml
<tx:annotation-driven transaction-manager="txManager"/>
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="get*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="find*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="query*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="list*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="search*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="select*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="count*" propagation="REQUIRED" read-only="true"/>
        <tx:method name="*" propagation="REQUIRED" rollback-for="java.lang.Throwable"/>
    </tx:attributes>
</tx:advice>
<aop:config proxy-target-class="true">
    <aop:advisor advice-ref="txAdvice"
                 pointcut="execution(* com.qiongsong..service..*.*(..))"
                 order="100"/>
</aop:config>
````

使用`SqlSessionTemplate`之后，`SqlSession`的事务管理会默认由Spring事务管理框架托管，不再需要手工干预。

>Spring的事务管理基于AOP实现，声明式事务管理可以管理到方法级别。一般而言切面定义在服务接口实现通常命名为`XXService`，`Mapper`无需定义事务切面的。

将XML加入到Spring上下文中，即可在Spring托管的Bean中引用到映射器接口。

````java
@Autowired
private AuthorMapper authorMapper;
````

### 最佳实践

Mybatis提供了很多的配置属性参数，可以根据不同的应用场景进行自定义配置。这个章节主要涉及到Mybatis的一些常见的配置选项，以及这些配置项背后的意义

- 首先从数据源开始，在任何时候，都应该优先选择数据源进行连接，DBCP、C3P0以及阿里巴巴开源的Druid，都是很好的选择。使用数据源的核心原因在于连接池的引入，创建和关闭`Connection`是代价高昂的操作，几乎所有的数据源都可以配置连接池用于复用`Connection`资源，同时进行资源管理。久经考验的连接池，几乎没有理由不使用。具体的连接池配置则可以参考对应`DataSource`的文档说明。
- 缓存配置，数据查询的缓存是一个老生常谈的问题，缓存可以提高性能基本得到了大部分人的认同。但是怎么合理地使用缓存，却需要根据不同的场景进行判断。不分青红皂白添加缓存，可能没有办法达到最好的效果。下面主要针对数据库操作对缓存的使用进行讨论。
	现在主流的缓存存储结构为KV键值对模式，并且大多数情况下会让缓存驻留在内存中。KV都可以为任意的类型，在Java体系中，一般使用K对象的`equals`方法进行比较，`equals`为`true`即表示命中缓存。在缓存的作用范围内，通过`get(Key)`的方法，获取得到缓存值，可以减少值的创建。特别是当创建对象需要很高成本的时候，使用缓存的作用更加明显。
	但是缓存也有一个很明显的问题，缓存的数据和即使数据不一定一致，这就涉及到缓存过期问题。如何控制好缓存的过期问题，设计合适的缓存策略（包括缓存更新，缓存穿透等）是缓存应用首先要考虑的一个点。
	其次，在分布式系统下，和缓存的过期策略相伴相随的是缓存的共享策略，特别是在同构系统下，用户的请求可能会被发送到同构集群中的任意一个服务节点，在多步骤多流程的业务操作（一个业务操作可能多次访问后台数据）中，如果同构系统中的缓存数据不一致，可能会导致数据的异常。这时候就会设计到缓存共享的问题，考虑使用独立于应用系统的集中式缓存管理，例如Redis、Memcache等。有了集中式的缓存，再加上原来驻留在应用中的本地缓存，一套比较复杂的系统，在缓存的使用上往往都会涉及成多级缓存的模式。需要根据业务的特点，制定缓存策略，有几个比较通用的原则：对于参与业务计算的热点数据并且不经常进行修改，并且是分布式部署，应该将缓存设置为全局缓存。并在可能发送业务修改的地方，设置更新缓存的功能。对于不需要参与业务计算，并且存在大量访问的时候，可以适当创建多级缓存机制，在并在多个层级的缓存中，创建合适的TTL保存周期性的数据同步。
	在Mybatis中内置二级缓存，都是默认开启
	- 一级缓存基于`SqlSession`，在同一个`SqlSession`中相同Sql会在`SqlSession`中的`PerpetualCache`缓存中获取。
	- 二级缓存基于命名空间（MapperXML中配置），可以跨越多个`SqlSession`共享。并可以通过配置设置定时过期的时间，以及大小限制。但是有一个地方需要特别注意，二级缓存基于命名空间创建也就是，一般情况下，多个`Mapper`之间的缓存数据是不相互影响的，同一个`Mapper`中，发生数据更改，可以由Mybatis自动完成缓存更新，但如果相同数据存储在不同的`Mapper`中，则可能由于二级缓存的启用，带来数据不一致的问题。

	在Mybatis-Spring中事务的管理交由Spring事务管理器完成，每一个Sql语句的执行完成后都会执行`sqlSession.close()`方法，因此在Mybatis-Spring体系中，一级缓存无法生效。而二级缓存因为可以跨越`SqlSession`存在因此不受影响。

- Mybatis在ORM这个基础功能，也提供了丰富的应用，通过`ResultMap`可以实现复杂的映射对照。在实际编程中，因为基本都是简单的POJO映射，更推荐使用`ResultType`，由Mybatis自动完成映射工作。
- 插件机制，大多数流行的框架都会围绕一个内核构建一套扩展机制。在Mybatis中则是围绕`SqlSession`构建，通过插件拦截器和调用链机制，完成拓展功能。








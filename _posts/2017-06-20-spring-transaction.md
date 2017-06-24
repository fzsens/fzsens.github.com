---
layout: post
title: Spring事务揭秘
date: 2017-06-20
categories: spring
tags: [spring,transaction]
description: Spring的事务处理过程分析
---

### Did it yourself

如在介绍Mybatis的几篇文章说到的，我们平常使用的事务是依赖于数据库来完成的，而在编程过程中使用最多的Spring事务也只是封装了和数据库交互的`Connection`从而进一步利用`Connection`来完成对不同事务的管理。假设由我们自己来实现一个事务，一个典型的例子就是两个独立的数据库操作，需要保证其在一个操作单元中，更确切点就是需要保证原子性。

````java
class ClassA {
  public void insert1(Object1 o) {
    // get connection
    // use connection do some use
    // commit
    // close connection
  }
}

class ClassB {
  public void insert2(Object2 o) {
    // get connection
    // do some insert
    // commit
    // close connection
  }
}
````

一个数据库操作过程一般四个过程构成，获取`Connection` ；进行SQL操作 ；提交；关闭链接，如果希望`insert1`和`insert2`在同一个`Transaction`中，很显然只要保证两个方法获取到的是同一个`Connection`即可，在两个方法都执行成功后进行提交，或者出现异常进行回滚，则可以通过数据库的undo机制保证事务特性。

如果是我们自己进行实现，遇到的第一个问题就是如何在两个方法之间传递`Connection`，能想到的一个办法是通过`ThreadLocal`来绑定线程变量，实现在不同方法上下文之间共享`Connection` ；第二个问题是如何控制提交和回滚。这两个问题都可以通过切面编程来解决。将方法进行增强，在方法开始之前获取`Connection`并绑定到`ThreadLocal`中，方法执行的时候`getConnection()`的操作，需要从刚才绑定的`ThreadLocal`中获取，完成操作之后，再由增强方法进行同意的提交。

在Spring中对事务的管理也大体按照上面的思路来实现。

### 框架概览

Spring的事务管理器，核心接口为`PlatformTransactionManager`

````java
public interface PlatformTransactionManager {
  /**
	 * 根据事务定义获取一个事务状态标示 
	 */
  TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
  /**
	 * 提交事务
	 */
  void commit(TransactionStatus status) throws TransactionException;
  /**
	 * 回滚事务
	 */
  void rollback(TransactionStatus status) throws TransactionException;

}
````

> `TransactionDefinition`定义了事务的属性信息，包括是否开启只读事务、超时、隔离级别、传播级别等，默认情况下使用`DefaultTransactionDefinition`即为开启读写事务、超时不设置、隔离级别不做设置、传播级别为`PROPAGATION_REQUIRED`，注意这边的属性设置，并非所有都能生效需要依赖具体的数据库和驱动，具体的事务属性意义可以在其他的网站中找到，这里稍微讲一下`readOnly`属性，其是对`connection.setReadOnly(boolean)`的封装，给对应的资源管理器一个优化提示，毕竟只读事务相比读写事务少了redo log等很多操作对性能提供有帮助，但是最后是否进行优化，依赖于具体的资源管理器（数据库）来决定。
>
> `TransactionStatus`记录了事务的状态，在提交或者回滚时候，根据事务状态实现不同的操作，其主要完成几个工作：查询事务状态；`setRollbackOnly()`用于标记当前事务使其回滚；SavePoint功能用于支持嵌套事务；

![PlatformTransactionManager](http://ooi50usvb.bkt.clouddn.com/PlatformTransactionManager.png)

从这个接口中就可以知道，事务管理的核心就是开启/获得事务，提交以及回滚事务。具体的实现依赖于各个事务管理平台。`PlatformTransactionManager`使用策略模式，在它提供的基础抽象的基础上，可以根据不同的数据访问层选择不同的实现策略。我们使用较多的为`DataSourceTransactionManager`,其为`JDBC`以及`Mybatis`提供事务管理服务。

Spring使用模板方法的设计模式，提供了`AbstractPlatformTransactionManager`作为所有的事务管理器的固有实现，预留一些模板方法给不同的实现类型进行定制实现。

````java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
  // template 设计模式
  Object transaction = doGetTransaction();
  // Cache debug flag to avoid repeated checks.
  boolean debugEnabled = logger.isDebugEnabled();
  if (definition == null) {
    // 使用默认的Transaction属性定义
    // Use defaults if no transaction definition given.
    definition = new DefaultTransactionDefinition();
  }
  // 当前是否存在事务
  if (isExistingTransaction(transaction)) {
    // Existing transaction found -> check propagation behavior to find out how to behave.
    return handleExistingTransaction(definition, transaction, debugEnabled);
  }
  // Check definition settings for new transaction.
  if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
    throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
  }
  // No existing transaction found -> check propagation behavior to find out how to proceed.
  if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
    // Mandatory && not existing 抛出错误
    throw new IllegalTransactionStateException(
      "No existing transaction found for transaction marked with propagation 'mandatory'");
  }
  else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
           definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
           definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
    // 在没有当前事务的情况下，都创建新的事务
    // 暂停无关的事务，TransactionSynchronizationManager 设置为暂停
    SuspendedResourcesHolder suspendedResources = suspend(null);
    if (debugEnabled) {
      logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
    }
    try {
      boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
      // 创建一个TransactionStatus
      DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
      /**
                 * template 方法，开启一个事务
                 */
      doBegin(transaction, definition);
      //绑定事务信息到当前线程中
      prepareSynchronization(status, definition);
      return status;
    }
    catch (RuntimeException ex) {
      resume(null, suspendedResources);
      throw ex;
    }
    catch (Error err) {
      resume(null, suspendedResources);
      throw err;
    }
  }
  else {
    // Create "empty" transaction: no actual transaction, but potentially synchronization.
    if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
      logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                  "isolation level will effectively be ignored: " + definition);
    }
    boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
    return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
  }
}
````

`getTransaction`方法的核心实际上是开启一个事务。

````java
private void processRollback(DefaultTransactionStatus status) {
  try {
    try {
      // 触发TransactionSynchronization的前置是件
      triggerBeforeCompletion(status);
      if (status.hasSavepoint()) {
        if (status.isDebug()) {
          logger.debug("Rolling back transaction to savepoint");
        }
        status.rollbackToHeldSavepoint();
      }
      else if (status.isNewTransaction()) {
        if (status.isDebug()) {
          logger.debug("Initiating transaction rollback");
        }
        // 模板方法，JDBC中 调用connection.rollback
        doRollback(status);
      }
      else if (status.hasTransaction()) {
        // 当前存在事务，并且rollback状态被设置
        if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
          if (status.isDebug()) {
            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
          }
          // JDBC中设置DataSourceTransactionObject为RollbackOnly
          doSetRollbackOnly(status);
        }
        else {
          if (status.isDebug()) {
            logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
          }
        }
      }
      else {
        logger.debug("Should roll back transaction but cannot - no transaction available");
      }
    }
    catch (RuntimeException ex) {
      triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
      throw ex;
    }
    catch (Error err) {
      triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
      throw err;
    }
    // 清理事务相关的Synchronizations
    triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
  }
  finally {
    // 事务事务资源，恢复之前的挂起的事务
    cleanupAfterCompletion(status);
  }
}
````

实际执行回滚，调用各个具体实现的`doRollback`方法，并完成一些后续的清理工作

````java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
  try {
    boolean beforeCompletionInvoked = false;
    try {
      // 触发synchronization
      prepareForCommit(status);
      triggerBeforeCommit(status);
      triggerBeforeCompletion(status);
      beforeCompletionInvoked = true;
      boolean globalRollbackOnly = false;
      if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
        globalRollbackOnly = status.isGlobalRollbackOnly();
      }
      if (status.hasSavepoint()) {
        if (status.isDebug()) {
          logger.debug("Releasing transaction savepoint");
        }
        // 处理嵌套事务
        status.releaseHeldSavepoint();
      }
      else if (status.isNewTransaction()) {
        if (status.isDebug()) {
          logger.debug("Initiating transaction commit");
        }
        // 模板方法 JDBC connection.commit();
        doCommit(status);
      }
      // Throw UnexpectedRollbackException if we have a global rollback-only
      // marker but still didn't get a corresponding exception from commit.
      if (globalRollbackOnly) {
        throw new UnexpectedRollbackException(
          "Transaction silently rolled back because it has been marked as rollback-only");
      }
    }
    // 触发TransactionSynchronization相关的事件
    catch (UnexpectedRollbackException ex) {
      // can only be caused by doCommit
      triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
      throw ex;
    }
    catch (TransactionException ex) {
      // can only be caused by doCommit
      if (isRollbackOnCommitFailure()) {
        doRollbackOnCommitException(status, ex);
      }
      else {
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
      }
      throw ex;
    }
    catch (RuntimeException ex) {
      if (!beforeCompletionInvoked) {
        triggerBeforeCompletion(status);
      }
      doRollbackOnCommitException(status, ex);
      throw ex;
    }
    catch (Error err) {
      if (!beforeCompletionInvoked) {
        triggerBeforeCompletion(status);
      }
      doRollbackOnCommitException(status, err);
      throw err;
    }

    // Trigger afterCommit callbacks, with an exception thrown there
    // propagated to callers but the transaction still considered as committed.
    try {
      triggerAfterCommit(status);
    }
    finally {
      triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
    }

  }
  finally {
    cleanupAfterCompletion(status);
  }
}
````

`AbstractPlatformTransactionManager`规定了整个事务管理的大体框架，一些具体的操作则交给派生类实现。

### ![AbstractPlatformTransactionManager](http://ooi50usvb.bkt.clouddn.com/AbstractPlatformTransactionManager3.png)

### 编程式事务管理

编码式的事务管理是Spring最基础的事务管理方式，只有理解了编码式的事务管理，才能知道Spring在整个事务管理中的角色和起到的作用。

直接使用Spring框架提供的原声API进行事务管理

````java
DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
TransactionStatus status = transactionManager.getTransaction(definition);
try {
  Connection conn = DataSourceUtils.getConnection(dataSource);
  PreparedStatement ps = conn.prepareStatement("insert into ROLE (DESCRIPTION, NAME) values (?, ?)");
  ps.setString(1, "abc");
  ps.setString(2, "889900");
  ps.executeUpdate();

  Connection conn2 = DataSourceUtils.getConnection(dataSource);
  PreparedStatement ps2 = conn2.prepareStatement("insert into USER (DESCRIPTION, NAME) values (?, ?)");
  ps2.setString(1, "4");
  ps2.setString(2, "4");
  ps2.executeUpdate();
} catch (Exception e) {
  transactionManager.rollback(status);
  throw  e;
}
transactionManager.commit(status);
````

这里核心是`Connection conn = DataSourceUtils.getConnection(dataSource);`

````java
public static Connection doGetConnection(DataSource dataSource) throws SQLException {
  Assert.notNull(dataSource, "No DataSource specified");

  // 从当前上下文中Connection
  ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
  if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
    conHolder.requested();
    if (!conHolder.hasConnection()) {
      logger.debug("Fetching resumed JDBC Connection from DataSource");
      conHolder.setConnection(dataSource.getConnection());
    }
    // 获取当前线程的Connection
    return conHolder.getConnection();
  }
  // Else we either got no holder or an empty thread-bound holder here.

  logger.debug("Fetching JDBC Connection from DataSource");
  // 新创建一个Connection
  Connection con = dataSource.getConnection();

  if (TransactionSynchronizationManager.isSynchronizationActive()) {
    logger.debug("Registering transaction synchronization for JDBC Connection");
    // Use same Connection for further JDBC actions within the transaction.
    // Thread-bound object will get removed by synchronization at transaction completion.
    ConnectionHolder holderToUse = conHolder;
    if (holderToUse == null) {
      holderToUse = new ConnectionHolder(con);
    }
    else {
      holderToUse.setConnection(con);
    }
    holderToUse.requested();
    // 注册 synchronization，主要是事务调用之后的清理回调
    TransactionSynchronizationManager.registerSynchronization(
      new ConnectionSynchronization(holderToUse, dataSource));
    holderToUse.setSynchronizedWithTransaction(true);
    if (holderToUse != conHolder) {
      TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
    }
  }

  return con;
}
````

当在同一个线程中获取线程的时候会获取同一个`Connection`。使用原生Spring API的方式，显然比较繁琐，在Spring中大量使用的一种设计模式是模板方法，例如各种XxxTemplate类，一般都是定义了一个执行模板，通过模板方法和策略模式结合实现设计上的解耦和拓展上的灵活。 同样的对于编程式事务的管理也提供了`TransactionTemplae`

````java
PlatformTransactionManager platformTransactionManager = new DataSourceTransactionManager(dataSource);
Object result = new TransactionTemplate(platformTransactionManager).execute(new TransactionCallback<Object>() {
  public Object doInTransaction(TransactionStatus transactionStatus) {
    int result = 0;
    try {
      Connection conn = DataSourceUtils.getConnection(dataSource);
      PreparedStatement ps = conn.prepareStatement("insert into ROLE (DESCRIPTION, NAME) values (?, ?)");
      ps.setString(1, "abc");
      ps.setString(2, "889900");
      result = ps.executeUpdate();

      Connection conn2 = DataSourceUtils.getConnection(dataSource);
      PreparedStatement ps2 = conn2.prepareStatement("insert into USER (DESCRIPTION, NAME) values (?, ?)");
      ps2.setString(1, "4");
      ps2.setString(2, "4");
      result = ps2.executeUpdate();

      Assert.assertEquals(1, result);
    } catch (SQLException e) {
      e.printStackTrace();
      // 设置回滚标识
      transactionStatus.setRollbackOnly();
    }
    return result;
  }
});
````

### 声明式事务管理 

编程式的事务管理，很显然灵活但是又非常繁琐，事务管理代码和业务操作代码混在一起，有没有办法能够对事务管理和业务操作进行解耦。从`TransactionTemplate`中可以得到一些启发，`TransactionTemplate`讲提交的逻辑封装在内部，如果可以为每一个事务的执行单元创建一个代理对象，讲这部分逻辑也封装起来是否就可以达到目的。实际上Spring的声明式事务管理正是通过这种方式实现。在具体实现声明式的事务管理的时候，Spring提供了多种方法有些是早期的`ProxyFactoryBean`+`TransactionInterceptor`，`TransactionFactoryBean`，`BeanNameAutoProxyCreator`等，但目前使用最多的应该是Spring2.x后引入的声明式配置方式。

````xml
<!--
<tx:annotation-driven transaction-manager="txManager"/>
-->

<!-- spring 托管事务 -->
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource"/>
</bean>

<!-- 使用声明式切面 -->
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

<!--托管事务在 service 和 manager 包下-->
<!--<aop:aspectj-autoproxy/>-->
<aop:config proxy-target-class="true">
  <aop:advisor advice-ref="txAdvice"
               pointcut="execution(* com.qiongsong..services..*.*(..)) or execution(* com.qiongsong..manager..*.*(..))"
               order="100"/>
</aop:config>
````

`txManager`就是`DataSourceTransactionManager`，用于进行JDBC的事务管理。重点在`<tx:advics />`在Spring的解析中实际初始化类为`TxAdviceBeanDefinitionParser`

````java
@Override
protected Class<?> getBeanClass(Element element) {
  return TransactionInterceptor.class;
}
````

`TransactionInterceptor`就是我们生成代理的对象

````java
@Override
public Object invoke(final MethodInvocation invocation) throws Throwable {
  // Work out the target class: may be {@code null}.
  // The TransactionAttributeSource should be passed the target class
  // as well as the method, which may be from an interface.
  Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

  // Adapt to TransactionAspectSupport's invokeWithinTransaction...
  return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
    @Override
    public Object proceedWithInvocation() throws Throwable {
      return invocation.proceed();
    }
  });
}

protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {
		// If the transaction attribute is null, the method is non-transactional.
		// 获得Transaction的配置属性，TransactionAttribute是TransactionDefinition的拓展
        // 可以通过rollbackOn针对一些特性的异常进行回滚或者不回滚
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
        // TransactionManager
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
            // 创建一个事务，txInfo内部持有TransactionStatus
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
                // 异常回滚
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
                // 清理Transaction
				cleanupTransactionInfo(txInfo);
			}
            // 执行事务提交
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
           // 省略
        }
	}
````

再通过`aop:advisor`织入到符合EL匹配到的类中，从而实现事务的管理。

再看看`@Transactional`基于注解的声明式事务管理，

````xml
<!--开启@Transactional注解支持-->
<tx:annotation-driven transaction-manager="txManager"/>
````

对应的Spring处理类

````java
public BeanDefinition parse(Element element, ParserContext parserContext) {
  registerTransactionalEventListenerFactory(parserContext);
  String mode = element.getAttribute("mode");
  if ("aspectj".equals(mode)) {
    // mode="aspectj"
    registerTransactionAspect(element, parserContext);
  }
  else {
    // mode="proxy"
    AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext);
  }
  return null;
}
````

分成两类，一个是aspectj 一个是使用`AopAutoProxyConfigurer`的方式

````java
public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
  AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);

  String txAdvisorBeanName = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME;
  if (!parserContext.getRegistry().containsBeanDefinition(txAdvisorBeanName)) {
    Object eleSource = parserContext.extractSource(element);

    // Create the TransactionAttributeSource definition.
    // 通过@Transactional 获得TransactionAttribute
    RootBeanDefinition sourceDef = new RootBeanDefinition(
      "org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");
    sourceDef.setSource(eleSource);
    sourceDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);

    // TransactionInterceptor 
    // Create the TransactionInterceptor definition.
    RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
    interceptorDef.setSource(eleSource);
    interceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registerTransactionManager(element, interceptorDef);
    interceptorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
    String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);

    // 织入切面
    // Create the TransactionAttributeSourceAdvisor definition.
    RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class);
    advisorDef.setSource(eleSource);
    advisorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
    advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
    if (element.hasAttribute("order")) {
      advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
    }
    parserContext.getRegistry().registerBeanDefinition(txAdvisorBeanName, advisorDef);

    CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
    compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
    compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
    compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, txAdvisorBeanName));
    parserContext.registerComponent(compositeDef);
  }
}
````

太阳底下没有新鲜事，实际上只是讲xml中声明的对象，自动注册到BeanFactory中。

### 最佳实践

1. 带事务方法，应该尽量只在增删改的方法，而常规查询方法尽量在事务外进行处理，如果必须在事务内处理，尽量独立未readOnly事务，提示数据库进行优化。
2. 带事务方法中不要出现比较耗时的处理程序，或者`RPC`调用，因为，事务中数据库连接是不会释放的，如果每个事务的处理时间都非常长，那么宝贵的数据库连接资源将很快被耗尽。
3. 使用注解或者使用声明式的切面，取决于团队的成熟度，如果团队技术能力较好，建议使用基于注解的方式，基于注解的方式可以时刻提醒进行事务的处理，也利于针对不同的方法提供最优的事务优化；而使用配置切面的方式，则更多基于约定的明明准则，配置的时候更多使用宁可错杀不可漏过的原则，同时也减少了事务概念在编程过程中的存在感。
---
layout: post
title: 快学Mybatis-内部原理
date: 2016-12-11
categories: mybatis
tags: [mybatis]
description: 介绍Mybatis主要行为的代码实现
---

>网上关于Mybatis源代码的解析的文章已经有很多，本章节不打算再赘述。而是从提问题的角度入手，回答Mybatis是怎么做的，以及为什么要这么做。

### 问题1：映射器怎么注入到Mybatis中

通过`Configuration`实例的`addMapper`方法注入`Mapper`类

````java
public <T> void addMapper(Class<T> type) {
  mapperRegistry.addMapper(type);
}
````

在`MapperRegistry`中

````java
MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
parser.parse();
````

在parser.parse()方法中

````java
public void parse() {
	String resource = type.toString();
	if (!configuration.isResourceLoaded(resource)) {
	  loadXmlResource();
	  configuration.addLoadedResource(resource);
	  assistant.setCurrentNamespace(type.getName());
	  parseCache();
	  parseCacheRef();
	  Method[] methods = type.getMethods();
	  for (Method method : methods) {
		try {
		  // issue #237
		  if (!method.isBridge()) {
			parseStatement(method);
		  }
		} catch (IncompleteElementException e) {
		  configuration.addIncompleteMethod(new MethodResolver(this, method));
		}
	  }
	}
	parsePendingMethods();
}
````

其中的核心方法是`loadXmlResource()`，在这个方法中，处理了从查找映射文件XML到将各个`Statement`转换为`MappedStatement`注册到`Configuration`的过程
主要的代码如下

````java
XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
//将XML文件流写入到Configuration中
xmlParser.parse();

//将XML文件流写入到Configuration中
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
  	//处理 mapper节点
    configurationElement(parser.evalNode("/mapper"));
	//防止重复加载
    configuration.addLoadedResource(resource);
	//绑定映射器到命名空间,防止spring通过映射器多次读取,并重复加载
    bindMapperForNamespace();
  }

  //处理不能正常构建的语句
  parsePendingResultMaps();
  parsePendingChacheRefs();
  parsePendingStatements();
}

private void configurationElement(XNode context) {
  
  //命名空间 namespace = com.sinoservices.mybatis.mapper.AuthorMapper
  String namespace = context.getStringAttribute("namespace");
  if (namespace == null || namespace.equals("")) {
	throw new BuilderException("Mapper's namespace cannot be empty");
  }
  builderAssistant.setCurrentNamespace(namespace);
	//处理缓存引用
  cacheRefElement(context.evalNode("cache-ref"));
	//处理缓存
  cacheElement(context.evalNode("cache"));
	//参数映射,已经不推荐使用
  parameterMapElement(context.evalNodes("/mapper/parameterMap"));
	//ResultMap映射处理
  resultMapElements(context.evalNodes("/mapper/resultMap"));
	//处理sql片段,保存到 sqlFragments
  sqlElement(context.evalNodes("/mapper/sql"));
	//构建 SQL语句
  buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
}
````

执行完成`addMapper`方法之后,实际完成了从`XML/Annotation` => `MappedStatement`的转换过程,并将转换后的结果注入到`Configuration`实例的`mappedStatements`中.
通过`configuration. getMappedStatement(String statementId);`可以在全局范围获取MS对象。

>Mybatis是围绕着`Configuration`来进行配置管理的，所有的和注册、配置相关的元数据`Meta`都会保存在`Configuration`对象中，这个对象也会在很多Mybatis的运行时和解析时上下文中存在和传递。使用这样的配置核心类有助于聚合配置项，特别是在ORM这种应用场景下，很多行为需要依赖于配置完成，使用这样的方式能有效减少编程时候的复杂度。

### 问题2：SqlSession如何获取映射器

与注册类似，映射器的获取也是通过`Configuration`来实现的。

通过`getMapper`方法,并传入映射器的类型获取

````java
AuthorMapper authorMapper = sqlSession.getMapper(AuthorMapper.class);
````

`SqlSession`实际委托给`Configuration`执行,在问题1 中,我们已经将`Mapper`注册到`Configuration`中,现在要做的,是从`Configuration`中取出

````java
public <T> T getMapper(Class<T> type) {
  return configuration.<T>getMapper(type, this);
}

//Configuration.java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
      return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}

````

在这边,我们可以发现,我们最终得到的是一个映射器的代理实现`MapperProxy`，这也解释了为什么在Spring容器中，我们可以进行依赖注入的原因。

>放射代理在Mybatis中的使用很广泛，通过创建代理类来拓展能够实现接口方法实现方式的自定义，也解耦了用户使用接口比如`Mapper`和和最终的代理类之间的绑定关系，在代理类的内部实现就可以实现很多高级的功能。放射代理会额外消耗一部分性能，通过缓存可以减少这部分消耗。

### 问题3：映射器如何执行SQL

在问题2中，我们已经得知映射器的实现是`MapperProxy`，`MapperProxy`实现了 `InvocationHandler`接口，最终的方法执行代码为

````java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  //不需要拦截的方法,Object对象的定义的方法
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
	  return method.invoke(this, args);
    } else if (isDefaultMethod(method)) {
	  return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  //最终的映射器执行
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  return mapperMethod.execute(sqlSession, args);
}
````

最终执行代码的地方为`MapperMehod`，每一个映射器接口对应到一个`MapperMethod`对象。

````java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
      //根据SQL的基本类型，分离参数并执行
    switch (command.getType()) {
      case INSERT: {
		  //分离参数，command.getName即为StatementID
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
    }
    return result;
}
````

在`execute`方法中，判断类型之后，委托给`sqlSession`。实际饶了一圈，真正完成SQL操作的还是`SqlSessioon`对象来完成。下面使用一个最常用的查询来分析整个SQL执行的过程，`SqlSession`的默认实现对象为`DefaultSqlSession`。

````java
//constructor
public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
  this.configuration = configuration;
  this.executor = executor;
  this.dirty = false;
  this.autoCommit = autoCommit;
}
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
  MappedStatement ms = configuration.getMappedStatement(statement);
  return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
````

在不做特殊配置下，`executor`使用`SimpleExecutor`，执行查询和ORM的工作就落到了这个实现上。其他的`Executor`只是在处理方式上的差异，完成的功能基本和`SimpleExecutor`一致。

````java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLExceptio {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
      //statement处理器
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //构建查询的Statement
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.<E>query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
````

`StatementHandler`封装了`Statement`执行器，默认情况使用`PreparedStatementHandler`对应到`PreparedStatement`。

在`PreparedStatement`中，执行`query`之后，将`statement`交给`DefaultResultSetHandler`类进行最后的ORM映射处理，并返回最终的结果集,完成查询。

````java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  return resultSetHandler.<E> handleResultSets(ps);
}
````

在`ps.execute()`；执行SQL完成之后调用`resultSetHandler`进行结果集处理。

>最终的执行还是需要使用JDBC的内置对象，在此之前的配置和解析都是为了`Statement`最终执行服务，通过接口和方法签名获取
>通过`PreparedStatement`能够减少SQL注入的风险。

### 问题4：结果集如何完成ORM映射到对象

在问题3中，`PreparedStatement`已经完成执行，并委托给`DefaultResultHandler`来进行处理， 具体的从`ResultSet`到对象的映射关系也在这边完成。

````java
final List<Object> multipleResults = new ArrayList<Object>();
int resultSetCount = 0;
ResultSetWrapper rsw = getFirstResultSet(stmt);
//得到当前语句执行的所有ResultMap,一般只会有一个
List<ResultMap> resultMaps = mappedStatement.getResultMaps();
int resultMapCount = resultMaps.size();
validateResultMapsCount(rsw, resultMapCount);
while (rsw != null && resultMapCount > resultSetCount) {
  //存在多个ResultMap循环处理
  ResultMap resultMap = resultMaps.get(resultSetCount);
  //处理结果集
  handleResultSet(rsw, resultMap, multipleResults, null);
  rsw = getNextResultSet(stmt);
  cleanUpAfterHandlingResultSet();
  resultSetCount++;
}
````

Mybatis可以支持多个`ResultMap`的映射方式，不过实际上一个`ResultMap`已经能满足绝大多数的需求

````java
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
            //*没有指定ResultHandler,使用默认
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
            //*开始处理
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
        //关闭结果集
      closeResultSet(rsw.getResultSet());
    }
  }
````

标记为`*`号的三行代码完成了ORM的核心功能。其中需要特别关注的是`ResultHandler`接口，这个接口允许自定义结果及的处理方式，灵活定义不同的返回结果。

````java
//处理单个ResultMap的映射
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
  DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
  //内存分页,全部查询出来之后,跳过offset行
  skipRows(rsw.getResultSet(), rowBounds);
  //遍历整个ResultSet记录
  while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
      //确定ResultMap
    ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
    //完成映射
    Object rowValue = getRowValue(rsw, discriminatedResultMap);
    //调用ResultHandler完成结果处理
    storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
  }
}
````

每一个查询对象都会调用`ResultHandler`实现的`handleResult()`方法。默认情况下不做任何处理保存到`list`中通过`multipleResults.add(defaultResultHandler.getResultList());`取出并保存到结果集中返回.完成整个映射过程。

>在ORM环节，也体现了配置到行为转换的灵活引用，如果需要自定义ORM的过程，可以通过自定义实现`ResultHandler`等接口并添加到Mybatis配置的方式来实现。

### 问题5：Mybatis如何进行缓存的管理

缓存在查询时候生效，一般也需要在修改或者删除新增的时候更新。

以查询为例, DefaultSqlSession执行

````java
  public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
````

在前面说到，Mybatis支持二级缓存机制，对应到不同的`executor`实现，`BaseExecutor`(普通的执行器, 包含一级缓存)，`CachingExecutor`(二级缓存)。

- 首先看一级缓存

````java
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }
````

根据`ms`和查询参数等信息，可以为每一个SQL生成为一个的`CacheKey`

````java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql     boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
        //查询本地缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
		//处理延迟加载
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
}
````

然后通过本地缓存`localCache`进行第一次的查询。保存一级缓存的地方就是这个`localCache`，对应的类型为`PerpetualCache`，具体的存储委托给一个`HashMap`实现。
在进行更新或者删除，需要是本地缓存失效直接调用

````java
public void clearLocalCache() {
  if (!closed) {
    localCache.clear();
    localOutputParameterCache.clear();
  }
}
````

由上面的分析可知，一级缓存只在同一个`SqlSession`中生效(`executor`和`localCache`和`SqlSession`都是一对一绑定)，因此不存在并发问题(可以使用简单的`Map`实现, 不需要进行同步)，需要最大化一级缓存,应该将查询操作和更改操作分离，这样能保证最高的缓存命中。

- 再看二级缓存

启用二级缓存之后，`Executor`实现类为`CachingExecutor`

````java
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
      // ms中绑定的Cache,即是key,同时也是存储缓存的对象
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
                //读取缓存
                // 存在并发风险,
                // 应该在此处增加读写锁,防止 read-writer 的问题
                // 但是实际上也没有办法完全杜绝更新数据,出现不一致性的情况
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
````

`tcm`是一个`TransactionalCacheManager`对象。在`tcm`中，又维护一个以`Cache`对象为`key`，`TransactionCache`为实现的`Map`集合，而`TransactionCache`内部又是通过委托`Cache`来保存和获取缓存对象，这样的委托机制，有利于整个缓存系统的拓展。例如在`TransactionCache`中，可以对缓存穿透进行预防，而不需要修改原有引用`Cache`的代码，是开闭原则的一个实践。

````java
  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }
  private TransactionalCache getTransactionalCache(Cache cache) {
    TransactionalCache txCache = transactionalCaches.get(cache);
    if (txCache == null) {
      txCache = new TransactionalCache(cache);
      transactionalCaches.put(cache, txCache);
    }
    return txCache;
  }
````

这样，通过共享`Cache`的方式，就可以在同一个`ms`(命名空间)中实现`Cache`共享。缓存的更新也是用过`Cache`作为值，调用`tcm`的`clear`方法完成。

>对Java并发比较熟悉的同学可能会发现二级缓存可能存在一个并发控制的问题，因为`Cache`是`ms`持有的对象，而ms是多个`SqlSession`共享的，一般情况下一个线程会对应到一个`SqlSession`， 因此可能出现下面这种情况

|Thread-1|Thread-2|
|---------------|----|
|getCache(cache1)=>读取缓存|getCache(cache1)=>读取缓存|
|putCache(cache1,record1)=>修改缓存|putCache(cache1,record2)=>修改数据|

>当多个线程并发修改数据的时候。数据的线程安全而在访问层没有进行有效的同步控制，则线程安全的任务就交给到缓存实现来完成。
>而在默认的情况下，`Cache`通过层层委托会由`PerpetualCache`实现，在`perpetucalCache`中则是将缓存保存的任务交给`HashMap`完成，我们都知道`HashMap`并没有提供线程安全的访问保护。因此使用默认的缓存提供的时候需要特别注意这一点. 当自定义缓存实现的时候,如果有并发控制, 也需要在`putObject`和`getObject`的时候,添加对应的并发控制.实际上在mybatis3.2.6之前,Mybatis默认是提供了读写锁控制.有兴趣可以查阅相关的代码.[https://github.com/mybatis/mybatis-3/commit/ddc48a7691f3925724a6059041c09b0af3996502](https://github.com/mybatis/mybatis-3/commit/ddc48a7691f3925724a6059041c09b0af3996502)

### 问题6：插件是怎么工作的

和Mybatis中的绝大多数配置一样，插件也可以注册到`Configuration`中，注册到Configuration中的插件会形成一个调用链：

````java
  public void addInterceptor(Interceptor interceptor) {
    interceptorChain.addInterceptor(interceptor);
  }
````

获得拦截调用链之后，依然是通过代理模式来实现的拦截处理。`StatementHandler`拦截

````java
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
      // statementHandler 添加插件拦截逻辑
      statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
````

`Executor`拦截

````java
 public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
        //默认为SimpleExecutor
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
````

在`interceptorChain`中将拦截器递归调用各个拦截的`plugin`方法

````java
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
````

在我们定义的插件中，一般都会实现`plugin`方法，执行类型

````java
  @Override
  public Object plugin(Object target) {
    //根据类型值包装需要拦截的类型对象
    if (target instanceof StatementHandler)
        return Plugin.wrap(target, this);
    else
        return target;
  }
  //拦截包装
  //target 为 executor 对象 或者 statementHandler 对象
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
````

实际返回的是一个`Plugin`的代理对象，在执行方法时候，调用插件的拦截方法。

````java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
          //调用拦截链
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
````

整个插件的执行过程是一个比较典型的AOP编程实践。


>到这边主要的Mybatis功能和代码分析基本完成了，Mybatis的代码比较简洁，但是注释较少，可能作者觉得逻辑比较简单，不需要额外太多注释。我fork了`3.4.2-snapshot`的版本，并添加了一些中文注释，可以在github中查看到，[mybatis-3 with comments](https://github.com/fzsens/mybatis-3)
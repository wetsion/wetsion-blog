---
title: Mybatis源码学习：一级缓存
description: Mybatis源码学习：一级缓存
categories:
 - 后端
 - Mybatis
tags:
 - Java
 - Mybatis
---



&emsp;&emsp;Mybatis提供了两种缓存，一种是一级缓存，一种是二级缓存，今天先学习下一级缓存。

一级缓存的作用域是sqlsession会话级别，同一个会话中，相同的`MapperStatement`（调用相同mapper的相同方法，上一篇mapper注册注入的文章说到过，mapper的方法将会解析生成对应的`MapperStatement`），传相同的参数，那么将会从缓存中查询，不会再查询数据库。

下面通过几个例子验证：

首先定义了一个mapper（两个方法的sql是一样的，后面将会说到为什么这样）：

```java
@Mapper
public interface FoodMapper {

    @Select("select * from food where id=#{id}")
    Food getById(@Param("id") Long id);

    @Select("select * from food where id=#{id}")
    Food selectById(@Param("id") Long id);
}
```

然后调用mapper：

```java
    @Autowired
    SqlSessionFactory sqlSessionFactory;

    @GetMapping("/mybatis/test2")
    public void test2() {
        SqlSession session = sqlSessionFactory.openSession();
        Food food = (Food) session.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        log.info(JSON.toJSONString(food));
        Food food2 = (Food) session.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        log.info(JSON.toJSONString(food2));
    }

    @GetMapping("/mybatis/test3")
    public void test3() {
        SqlSession session = sqlSessionFactory.openSession();
        Food food = (Food) session.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        log.info(JSON.toJSONString(food));
        Food food2 = (Food) session.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.selectById", 1L);
        log.info(JSON.toJSONString(food2));
    }

    @GetMapping("/mybatis/test4")
    public void test4() {
        SqlSession session = sqlSessionFactory.openSession();
        SqlSession session2 = sqlSessionFactory.openSession();
        Food food = (Food) session.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        log.info(JSON.toJSONString(food));
        Food food2 = (Food) session2.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        log.info(JSON.toJSONString(food2));
    }
```

> 可以看到，我使用`sqlSessionFactory`来手动调用`openSession`创建会话，为什么不直接注入foodMapper呢，其实上一篇mapper注册与注入的文章，我已经知道，其实使用mapper调用方法，实际上是通过MapperProxy代理对象创建`MapperMethod`对象并调用`execute()`方法，而`execute()`调用`SqlSessionTemplate`的对应方法，`SqlSessionTemplate`则创建代理会话sqlSessionProxy处理。所以我们每次使用`foodMapper.getById`都会创建一个会话，就不满足一级缓存的条件了。

 - 在test2()中，我在一个会话中，两次调用了`getById()`方法，并都传入参数1，结果如下：

```xml
Cache Hit Ratio [site.wetsion.mybatislearning.mapper.FoodMapper]: 0.0
JDBC Connection [HikariProxyConnection@173171954 wrapping com.mysql.cj.jdbc.ConnectionImpl@4959bc8] will not be managed by Spring
==>  Preparing: select * from food where id=? 
==> Parameters: 1(Long)
<==    Columns: id, name, color
<==        Row: 1, aa, red
<==      Total: 1
2019-06-27 22:52:54.135  INFO 30524 --- [nio-9066-exec-3] s.w.mybatislearning.web.FoodController   : {"color":"red","id":1,"name":"aa"}
Cache Hit Ratio [site.wetsion.mybatislearning.mapper.FoodMapper]: 0.0
2019-06-27 22:52:54.137  INFO 30524 --- [nio-9066-exec-3] s.w.mybatislearning.web.FoodController   : {"color":"red","id":1,"name":"aa"}
```

可以看到，只打印了一条查询sql，只查询了一次，第二次从缓存中读取的

 - 在test3()中，在同一个会话中，我分别调用`getById`和`selectById`，参数都为1，上面定义也知道，这两个方法执行的语句是一样的，运行结果如下：

```java
Cache Hit Ratio [site.wetsion.mybatislearning.mapper.FoodMapper]: 0.0
JDBC Connection [HikariProxyConnection@377160664 wrapping com.mysql.cj.jdbc.ConnectionImpl@18bcff67] will not be managed by Spring
==>  Preparing: select * from food where id=? 
==> Parameters: 1(Long)
<==    Columns: id, name, color
<==        Row: 1, aa, red
<==      Total: 1
2019-06-27 22:56:38.756  INFO 30524 --- [nio-9066-exec-5] s.w.mybatislearning.web.FoodController   : {"color":"red","id":1,"name":"aa"}
Cache Hit Ratio [site.wetsion.mybatislearning.mapper.FoodMapper]: 0.0
==>  Preparing: select * from food where id=? 
==> Parameters: 1(Long)
<==    Columns: id, name, color
<==        Row: 1, aa, red
<==      Total: 1
2019-06-27 22:56:38.759  INFO 30524 --- [nio-9066-exec-5] s.w.mybatislearning.web.FoodController   : {"color":"red","id":1,"name":"aa"}
```

可以看到，执行了两次sql查询语句，且都一样，说明不满足一级缓存的条件.

> 网上有的文章说，查询语句相同的sql会被缓存，这里验证了，查询语句相同并不是必要条件


 - 在test4()中，我创建了两个会话，调用相同的方法，结果就不用贴出来了，依然是执行了两次sql查询，并没使用缓存。  


---

上面的例子证明了一级缓存确实是在同一会话，相同的MapperStatement，相同的参数，则会使用缓存。那么原理是什么呢，还是探索分析以下源码。

从上面的selectOne()入手，我们看下`DefaultSqlSession#selectOne()`（上篇文章已知道mybatis实际默认使用就是`DefaultSqlSession`）:

```java
public <T> T selectOne(String statement, Object parameter) {
    List<T> list = this.selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }
```

额。。继续看selectList()：

```java
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
```

根据mapper中方法的全限定名从`Configuration`中获取对应的`MapperStatement`对象，然后再调用executor的`query()`方法。

这个executor类型是`Executor`接口，那么实际是哪个实现类呢，executor属性是通过`DefaultSqlSession`构造方法设置的，而创建`DefaultSqlSession`是通过`DefaultSqlSessionFactory#openSession()`，再调用`openSessionFromDataSource()`：

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

可以看到，executor来自`Configuration`类：

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

默认是`SimpleExecutor`，但由于默认`cacheEnabled`属性为true，所以通过装饰器模式对`SimpleExecutor`进行了包装，真正使用的是`CacheExecutor`。紧接上文，看一下`CacheExecutor`的`query()`：

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

`BundleSql`呢，就是持有实际的sql字符串，以及一些参数，而`CacheKey`，顾名思义，就是缓存的键。即生成了sql和缓存键之后，调用query()，似乎要开始真正的查询操作：

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

`Cache cache = ms.getCache()`这里是获取二级缓存，由于这里没设置二级缓存，暂不提，继续看，调用了delegate.query()，这里delegate就是上文说的默认的`SimpleExecutor`，而`SimpleExecutor`自身没有query()，继承了父类`BaseExecutor`的query()，所以一级缓存的核心就是`BaseExecutor`的query()：

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
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
```

可以看到这样一行`list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;`即从本地缓存localCache中获取，这个locaCache是BaseExecutor中的属性，PerpetualCache对象，就是一级缓存，内部实现其实就是一个hashMap，如果这个list结果为空，就会执行`list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);`：

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

这是真正的从数据库查询，查询后再往一级缓存localCache中放一份。

到这里，就已经知道一级缓存查询的原理了，那为什么当执行更新操作时，就会清除一级缓存呢，也顺带把这个也看一下。

同理，先看`DefaultSqlSession`的update方法：

```java
@Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

大同小异，从上面我们已经知道实际是CacheExecutor，所以看CacheExecutor的update：

```java
@Override
  public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    flushCacheIfRequired(ms);
    return delegate.update(ms, parameterObject);
  }
```

这里`flushCacheIfRequired(ms)`是和二级缓存相关，暂不看，从上面我们也知道，delegate是SimpleExecutor，但SimpleExecutor的方法都继承自`BaseExecutor`，所以直接看`BaseExecutor`的update：


```java
@Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
  @Override
  public void clearLocalCache() {
    if (!closed) {
      localCache.clear();
      localOutputParameterCache.clear();
    }
  }
```

从方法名我们也知道了，`clearLocalCache()`如果会话没被关闭，清空一级缓存，然后再执行数据库操作。



至此，一级缓存读写的原理在梳理源码的过程中已经很清晰了，之后再学习下二级缓存原理。
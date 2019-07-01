---
title: Mybatis源码学习：二级缓存
description: Mybatis源码学习：二级缓存
categories:
 - 后端
 - Mybatis
tags:
 - Java
 - Mybatis
---



上一篇文章说到了mybatis的一级缓存，这篇继续学习mybatis的二级缓存。

二级缓存不同于一级缓存只作用于同一会话中，而是全局的，或者说，对于不同的会话sqlsession，只要是同一个mappedstatement（即相同的mapper同一个方法），就会生效（前提是sqlsession提交之后）。

打开二级缓存，除了默认打开的`mybatis.configuration.cache-enabled`属性之外，还需要在需要的mapper接口上使用`@CacheNamespace`注解（等价于xml配置时在mapper xml文件里加上`<cache/>`标签）。

### 示例实践

例如以下：

```java
@Mapper
@CacheNamespace
public interface FoodMapper {

    @Select("select * from food where id=#{id}")
    Food getById(@Param("id") Long id);
}
```

然后在controller中调用（这里还是和上一篇一级缓存一样用的是通过sessionFactory创建会话，其实也可以直接注入foodMapper调用）：

```java
@GetMapping("/mybatis/test4")
    public void test4() {
        SqlSession session = sqlSessionFactory.openSession();
        SqlSession session2 = sqlSessionFactory.openSession();
        Food food = (Food) session.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        session.commit();
        log.info(JSON.toJSONString(food));
        Food food2 = (Food) session2.selectOne(
                "site.wetsion.mybatislearning.mapper.FoodMapper.getById", 1L);
        log.info(JSON.toJSONString(food2));
    }
```

以上的demo开启了两个会话，共执行了两次查询操作，但只打印了一次sql查询语句，但如果把`session.commit()`删除，那么将会打印两次sql查询语句，这也是前面说的二级缓存在会话提交之后生效。

### 源码学习

那么还是通过从源码学习下二级缓存的原理，同样的，先从`DefaultSqlSession#selectOne()`入手，上一篇文章已经知道，`DefaultSqlSession`实际上先调用的是`CachingExecutor`的`query()`：

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

在上述源码中，`Cache cache = ms.getCache()`即为获取二级缓存。

---

***那么在mappedStatement中的cache是何时初始化设置与`MappedStatement`绑定的呢***

还是得从mapper bean的创建与注入说起，[上一篇文章](https://www.wetsion.site/mybatis-mapper-register-autowire.html)我知道了mapper的bean通过`MapperFactoryBean`创建，而在`MapperFactoryBean`初始化的时候调用`Configuration#addMapper`，而`Configuration#addMapper`调用`MapperRegistry#addMapper()`，在这里将mapper包装成了`MapperProxyFactory`放入缓存中，着重看下这个方法：

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

上一篇文章就说到，`MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type); parser.parse();`是将mapper中的方法解析成对于的MappedStatement对象。在`MapperAnnotationBuilder`的构造方法中，对assistant属性进行了初始化（assistant是`MapperBuilderAssistant`对象，后面会通过这个助手类创建cache），再看下`parse()`这个方法：

```java
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
```

其中，对`assistant`助手类设置了命名空间（这个namespace后面将用于作为cache的ID），`parseCache()`就是判断mapper接口是否有`@CacheNamespace`注解修饰，如果有，则接入二级缓存：

```java
private void parseCache() {
    CacheNamespace cacheDomain = type.getAnnotation(CacheNamespace.class);
    if (cacheDomain != null) {
      Integer size = cacheDomain.size() == 0 ? null : cacheDomain.size();
      Long flushInterval = cacheDomain.flushInterval() == 0 ? null : cacheDomain.flushInterval();
      Properties props = convertToProperties(cacheDomain.properties());
      assistant.useNewCache(cacheDomain.implementation(), cacheDomain.eviction(), flushInterval, size, cacheDomain.readWrite(), cacheDomain.blocking(), props);
    }
  }
```

通过`assistant`助手类对象，创建一个新的cache对象：

```java
public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
```

前面说到，namespace作为cache的ID，这在CacheBuilder的构造方法中进行了设置：

```java
public CacheBuilder(String id) {
    this.id = id;
    this.decorators = new ArrayList<>();
  }
```

通过CacheBuilder对象，进行一系列属性配置，最后通过`build()`方法创建Cache对象，将属性装配到Cache对象中，再返回：

```java
public Cache build() {
    setDefaultImplementations();
    Cache cache = newBaseCacheInstance(implementation, id);
    setCacheProperties(cache);
    if (PerpetualCache.class.equals(cache.getClass())) {
      for (Class<? extends Cache> decorator : decorators) {
        cache = newCacheDecoratorInstance(decorator, cache);
        setCacheProperties(cache);
      }
      cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
      cache = new LoggingCache(cache);
    }
    return cache;
  }
```

再回到`MapperBuilderAssistant#useNewCache()`，创建完cache对象之后，通过`configuration.addCache(cache)`将cache对象存到`Configuration`的`caches`这样一个map中，并在当前的`MapperBuilderAssistant`对象assistant的currentCache属性保存一份（在后面会用到）。

cache对象已经创建好了，那么是怎么装配到mapper接口中方法对应的MappedStatement对象中的呢，接着看`MapperAnnotationBuilder#parse()`方法，在`parseCache()`之后，foreach遍历所有的方法，然后通过`parseStatement()`解析生成MappedStatement对象：

```java
void parseStatement(Method method) {
    Class<?> parameterTypeClass = getParameterType(method);
    LanguageDriver languageDriver = getLanguageDriver(method);
    SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
    if (sqlSource != null) {
      Options options = method.getAnnotation(Options.class);
      final String mappedStatementId = type.getName() + "." + method.getName();
      Integer fetchSize = null;
      Integer timeout = null;
      StatementType statementType = StatementType.PREPARED;
      ResultSetType resultSetType = null;
      SqlCommandType sqlCommandType = getSqlCommandType(method);
      boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
      boolean flushCache = !isSelect;
      boolean useCache = isSelect;
      KeyGenerator keyGenerator;
      String keyProperty = null;
      String keyColumn = null;
      if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
        // first check for SelectKey annotation - that overrides everything else
        SelectKey selectKey = method.getAnnotation(SelectKey.class);
        if (selectKey != null) {
          keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
          keyProperty = selectKey.keyProperty();
        } else if (options == null) {
          keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        } else {
          keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
          keyProperty = options.keyProperty();
          keyColumn = options.keyColumn();
        }
      } else {
        keyGenerator = NoKeyGenerator.INSTANCE;
      }
      if (options != null) {
        if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
          flushCache = true;
        } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
          flushCache = false;
        }
        useCache = options.useCache();
        fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
        timeout = options.timeout() > -1 ? options.timeout() : null;
        statementType = options.statementType();
        resultSetType = options.resultSetType();
      }
      String resultMapId = null;
      ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
      if (resultMapAnnotation != null) {
        resultMapId = String.join(",", resultMapAnnotation.value());
      } else if (isSelect) {
        resultMapId = parseResultMap(method);
      }
      assistant.addMappedStatement(
          mappedStatementId,
          sqlSource,
          statementType,
          sqlCommandType,
          fetchSize,
          timeout,
          // ParameterMapID
          null,
          parameterTypeClass,
          resultMapId,
          getReturnType(method),
          resultSetType,
          flushCache,
          useCache,
          // TODO gcode issue #577
          false,
          keyGenerator,
          keyProperty,
          keyColumn,
          // DatabaseID
          null,
          languageDriver,
          // ResultSets
          options != null ? nullOrEmpty(options.resultSets()) : null);
    }
  }
```

该方法中，先根据方法的注解和方法传参获取sql资源，即判断是否有`@Insert`、`@Update`、`Select`、`@Delete`注解，如果有这些注解，获取注解中的sql语句，根据注解中的sql语句和传参创建SqlSource对象。

- 如果SqlSource对象不为空

    - 再获取方法上`@Options`注解（`@Options`注解可以配置是否使用缓存、是否主键自增、主键的列名等属性）
    
    - 再根据sql类型创建不同的主键生成器`KeyGenerator`
    
        - 如果是Insert和Update类型，再获取方法上的`@Selectkey`注解
        
            - 如果`@SelectKey`注解存在，根据selectKey注解也创建一个MapperStatement对象，它的id是原方法对应的MapperStatement对象ID再加上`!selectKey`后缀（原MapperStatement对象ID为类全限定名加上`.`加上方法名），调用助手类***assistant.addMappedStatement***创建（暂放在后面说），并创建`SelectKeyGenerator`放入`Configuration`的keyGenerators中
            
            - 如果`@SelectKey`注解不存在，则按照`@Options`注解配置的或者`Configuration`中配置的
        
        - 如果是非Insert和Update，使用`NoKeyGenerator`
        
    - 如果`@Options`注解存在，获取该注解的属性，覆盖默认配置
    
    - 调用当前助手类`MapperBuilderAssistant`对象assistant的`addMappedStatement()`方法
    
    
着重看一下`MapperBuilderAssistant#addMappedStatement()`，该方法主要用于创建MappedStatement，并通过`cache(currentCache)`，将上面步骤中`MapperBuilderAssistant#useNewCache()`创建并放入当前MapperBuilderAssistant对象的cache，与创建的MappedStatement绑定，再将`MappedStatement`对象放入到`Configuration`的mappedStatements中保存。

至此，关于`MappedStatement`中cache的来源以及cache如何和`MappedStatement`绑定的，就已经明了了。

---

回到文章前面，看到的`CachingExecutor`的`query()`的地方，通过`Cache cache = ms.getCache()`获取缓存对象cache


- 如果cache存在，并且isUseCache，则按照二级缓存情况处理。
 
    - 先通过`TransactionalCacheManager`对象`tcm.getObject(cache, key)`方法根据cache获取对应的`TransactionalCache`进而再根据key去`TransactionalCache`中的Cache对象获取缓存，这里key是`CacheKey`对象，是根据当前方法对应的MappedStatement对象和方法传参以及sql等构建的一个对象，可用来唯一确定查询。

    - 如果通过`tcm.getObject(cache, key)`获取的缓存为空，则调用`BaseExecutor#query()`获取查询结果，再将查询结果通过`tcm.putObject(cache, key, list);`放入二级缓存  
    
    - 结果返回


- 如果cache不存在，则如同上篇一级缓存的文章说的，调用`BaseExecutor`的`query()`查询，返回结果

<br/>

&emsp;&emsp;`TransactionalCacheManager`是一个`TransactionalCache`的管理类，内部维护了一个`Cache`:`TransactionalCache`的键值对的map。而`TransactionalCache`是对Cache对象包装了一层，这个Cache对象就是`MappedStatement`中的cache，所以当`TransactionalCacheManager.getObject()`时，本质上还是从Cache中获取，那还包装一层`TransactionalCache`是不是有些多余呢，并不是，而是为了put操作，这也是接下来解释前文说的，为什么缓存在sqlSession会话提交之后才生效。

看一下`TransactionalCache#putObject()`和`#getObject()`：

```java
  @Override
  public Object getObject(Object key) {
    Object object = delegate.getObject(key);
    if (object == null) {
      entriesMissedInCache.add(key);
    }
    if (clearOnCommit) {
      return null;
    } else {
      return object;
    }
  }
  @Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
  }
```

可以看到，我在读缓存时，是从`delegate`（Cache对象）中获取key对应的值，而写入时，是往`entriesToAddOnCommit`中put。`entriesToAddOnCommit`是什么呢，是一个HashMap，可以理解为是一个待提交保存的集合，由此可见，在写缓存时并没有直接写入Cache。那么再看`TransactionalCache#commit()`：

```java
  public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
  }
  private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
  }
  private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }
```

可以看到，当进行会话提交时，会将待提交集合entriesToAddOnCommit中的数据迁移至Cache对象中，然后再将entriesToAddOnCommit清空。

至此，二级缓存的实现就已经了解了，结合上一篇一级缓存的学习，可以发现，使用二级缓存之后，查询数据，先尝试从二级缓存中拿数据，没有则去一级缓存中拿，一级缓存中再没有，才去查询数据库。


>补充：  
>对前文创建Cache做一些补充，其实返回Cache对象是被包装了很层的Cache对象，以`@CacheNamespace`的默认配置来看，在创建对象时，这层层包装的最底层是`PerpetualCache`，然后被`LruCache`包装，`LruCache`再被`ScheduledCache`包装，`ScheduledCache`再被`SerializedCache`包装，`SerializedCache`再被`LoggingCache`包装，`LoggingCache`再被`SynchronizedCache`包装，默认最终返回的是`SynchronizedCache`对象。

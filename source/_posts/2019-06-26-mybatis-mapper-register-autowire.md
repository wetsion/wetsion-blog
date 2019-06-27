---
title: Mybatis源码学习：mapper的注册与注入
description: Mybatis源码学习：mapper的注册与注入
categories:
 - 后端
 - Mybatis
tags:
 - Java
 - Mybatis
---


> 一直在使用mybatis，对它的源码断断续续地看过一点，知之甚少，所以打算把mybatis源码坚持看一遍，并记录成一系列学习笔记，以免后面忘了，可以看着再回忆回忆。

先从最常用的点入手，在使用的时候，通过都是定义一个mapper接口，例如这样：

```java
public interface FoodMapper {

    @Select("select * from food where id=#{id}")
    Food getById(@Param("id") Long id);
}
```

然后在启动类上加上mapper扫描路径：

```java
@SpringBootApplication
@MapperScan(basePackages = "site.wetsion.mybatislearning.mapper")
public class MybatisLearningApplication {

    public static void main(String[] args) {
        SpringApplication.run(MybatisLearningApplication.class, args);
    }

}
```

最后再在用到的地方注入使用：

```java
@RestController
@Slf4j
public class FoodController {

    @Autowired
    FoodMapper foodMapper;

    @GetMapping("/mybatis/test1")
    public void test() {
        Food food = foodMapper.getById(7L);
        log.info(JSON.toJSONString(food));
    }
}
```

这样一波操作看起来很简单，但都是mybatis和spring帮我做了很多事，所以先从这里入手，通过学习源码了解下是怎么样做到将mapper接口注册成bean，并再注入使用的。

首先，可以明确的一点是，是我通过使用`@MapperScan`注解告诉了spring，mapper接口的所在位置，所以先看该注解的源码（以下源码版本为jdk1.8，mybatis-spring-boot-starter  2.0.1）：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {

  // 省略注解的属性

}
```

这个注解通过Import引入了`MapperScannerRegistrar`，顾名思义，我先猜测它就是mapper的注册器，同时看到注解的注释有这样一句话：`It performs when same work as {@link MapperScannerConfigurer} via {@link MapperScannerRegistrar}.`，意思就是`MapperScannerRegistrar`和`MapperScannerConfigurer`做的是同样的工作，而`MapperScannerConfigurer`是干嘛的呢，使用xml配置时，通常这样指定mapper包路径，将mapper注册成bean：

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="site.wetsion.mybatislearning.mapper" />
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>
```

所以点开`MapperScannerRegistrar`，看它如何实现的：

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {
    @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
      registerBeanDefinitions(mapperScanAttrs, registry);
    }
  }
}
```

通过import引入`MapperScannerRegistrar`之后，执行完aware接口的操作之后，将会调用上述核心的`registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)`，可以看到，在该方法中，先获取`@MapperScan`注解的属性，不为空，将会调用私有的registerBeanDefinitions()方法，如下：

```java
void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry) {

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    Optional.ofNullable(resourceLoader).ifPresent(scanner::setResourceLoader);

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBeanClass(mapperFactoryBeanClass);
    }

    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("value"))
            .filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("basePackages"))
            .filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(
        Arrays.stream(annoAttrs.getClassArray("basePackageClasses"))
            .map(ClassUtils::getPackageName)
            .collect(Collectors.toList()));

    scanner.registerFilters();
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }
```

这里定义了`ClassPathMapperScanner`对象scanner，然后获取注解属性，并将属性设置到scanner中，最后调用doScan开始扫描指定的包路径：

```java
@Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```

先调用父类也就是`ClassPathBeanDefinitionScanner`的doScan方法，将指定包下的mapper接口转换成一个个bean定义持有者（BeanDefinitionHolder，拥有名称别名和bean定义，用于后续bean注册），并调用`BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry)`注册bean定义，再将beanDefinitionHolder返回。如果不为空，将调用processBeanDefinitions对这些beanDefinitionHolder处理：

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
          + "' and '" + beanClassName + "' mapperInterface");

      // mapper接口是bean的原始类，但bean的实际类是MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      definition.setBeanClass(this.mapperFactoryBeanClass);

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```

这里对beanDefinition进行一系列改造，包括<b>将beanClass设置为`MapperFactoryBean`</b>，此时，mapper接口是bean的原始类，但bean的实际类是MapperFactoryBean，这里将在后面注入mapper时体现。

这样，所有的mapper就注册进了Spring容器，上面说的是通过`@MapperScan`注解指定mapper的包路径的方式，通常我们也会使用另一种方式，即不在启动类使用`@MapperScan`注解，而是在每一个mapper接口上使用`@Mapper`注解来声明这是一个mapper，然而对于这种方式，是怎样注册成bean的呢。

可以在`MybatisAutoConfiguration`类中发现一个内部静态类`AutoConfiguredMapperScannerRegistrar`，这个类在另一个内部静态类，同时也声明为配置类的`MapperScannerRegistrarNotFoundConfiguration`引入：

```java
@org.springframework.context.annotation.Configuration
  @Import({ AutoConfiguredMapperScannerRegistrar.class })
  @ConditionalOnMissingBean(MapperFactoryBean.class)
  public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
      logger.debug("No {} found.", MapperFactoryBean.class.getName());
    }
  }
```

而`AutoConfiguredMapperScannerRegistrar`所做的和前文的`MapperScannerRegistrar`做的是类似的事：

```java
public static class AutoConfiguredMapperScannerRegistrar
      implements BeanFactoryAware, ImportBeanDefinitionRegistrar, ResourceLoaderAware {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      if (!AutoConfigurationPackages.has(this.beanFactory)) {
        logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.");
        return;
      }

      logger.debug("Searching for mappers annotated with @Mapper");

      List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
      if (logger.isDebugEnabled()) {
        packages.forEach(pkg -> logger.debug("Using auto-configuration base package '{}'", pkg));
      }

      ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
      if (this.resourceLoader != null) {
        scanner.setResourceLoader(this.resourceLoader);
      }
      scanner.setAnnotationClass(Mapper.class);
      scanner.registerFilters();
      scanner.doScan(StringUtils.toStringArray(packages));

    }
  }
```

即对根目录路径下所有类进行扫描，将有`@Mapper`注解修饰的接口注册成bean。


以上就是mapper接口注册到Spring容器的分析，下面再探索下当我们使用@Autowired尝试注入时，都做了什么事。

上文已经说了，在扫描注册beanDefinition之后，有一步将bean的class改成了MapperFactoryBean，而`MapperFactoryBean`实现了`FactoryBean`接口，实现此接口的bean不能用作普通的bean，它是对象的工厂，不能直接作为bean的实例，Spring在处理这类bean时，会调用`getObject()`获取bean对象。

同时`MapperFactoryBean`继承自`SqlSessionDaoSupport`，而`SqlSessionDaoSupport`继承`DaoSupport`，

> `DaoSupport`定义了初始化的方法，`SqlSessionDaoSupport`定义了设置数据访问的方式，如setSqlSessionTemplate、setSqlSessionFactory（这两个方法本质上都是为了给属性`SqlSessionTemplate sqlSessionTemplate`赋值），所以`MapperFactoryBean`在创建对象时需要设置这两个值其中一个，如果都设置了，setSqlSessionFactory中逻辑将会被覆盖。

`DaoSupport`实现了`InitializingBean`接口，所以在Spring容器通过反射创建`MapperFactoryBean`实例后，将会调用`afterPropertiesSet()`：

```java
@Override
	public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
		checkDaoConfig();
		try {
			initDao();
		}
		catch (Exception ex) {
			throw new BeanInitializationException("Initialization of DAO failed", ex);
		}
	}
```

随即调用本类的`checkDaoConfig()`:

```java
@Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();
    notNull(this.mapperInterface, "Property 'mapperInterface' is required");
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
```

通过调用`Configuration`的addMapper调用`MapperRegistry.addMapper()`，将当前bean的原始类（该mapper接口）包装成`MapperProxyFactory`对象保存至map缓存起来，如下所示：

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

上面还有一个地方比较重要：`MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);parser.parse();` 我再看下parse()这个方法：

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

这里主要是遍历mapper中的方法，将方法解析生成对应的`MapperStatement`对象，并保存在`Configuration`的`Map<String, MappedStatement> mappedStatements`中，具体实现见`MapperBuilderAssistant#addMappedStatement()`暂且不表。


衔接上文说的，`MapperFactoryBean`实例创建之后，在注入时，通过调用`getObject()`方法获取bean对象：

```java
@Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

这里getSqlSession获取的是`SqlSessionTemplate`，即`SqlSessionTemplate#getMapper()`：

```java
@Override
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }
```

其实就是把上面说的放入MapperRegistry的map缓存中的`MapperProxyFactory`对象取出来，再创建一个当前mapper接口的代理类对象(`MapperProxy`)，最后Spring将这个代理类对象注入到引用的地方：

```java
// MapperRegistry.java
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
// MapperProxyFactory.java
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

由于实际注入的是`MapperProxy`对象，所以当我`foodMapper.getById(1)`这样调用时，实际上调用的是`MapperProxy#invoke()`：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

将mapper的方法封装成`MapperMethod`对象，调用`MapperMethod`的execute方法，而在execute本质上是根据操作类型（insert、update、delete、select等）调用`SqlSessionTemplate`的对应方法，而`SqlSessionTemplate`通过内部的代理会话`sqlSessionProxy`进行数据库操作，获取结果。

到这里，就看完了mapper注册与注入的流程。


关于mybatis与Spring整合，mapper的注册与注入，就学习到这里，还有一些瑕疵，后续再写笔记进行补充。


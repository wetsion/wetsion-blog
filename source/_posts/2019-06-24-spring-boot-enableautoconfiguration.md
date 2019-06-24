---
title: SpringBoot：@EnableAutoConfiguration自动配置原理
description: 探索springboot的@EnableAutoConfiguration自动配置原理
categories:
 - 后端
 - Spring
tags:
 - Java
 - spring
---


&emsp;&emsp;众所周知，springboot的出现为开发带来了很大的方便，主要就是提供了很多自动配置，免去了开发人员花在配置上的时间，使我们开发者能更专注于业务或者架构设计上。

&emsp;&emsp;而springboot的自动配置是通过一系列注解来完成的，例如`@EnableAutoConfiguration`、`@Conditional`、`@ConditionalOn*`、`@EnableConfigurationProperties`等等。

&emsp;&emsp;这一系列注解我觉得大致可分为两类，一类是条件注解，一类是与配置相关的注解，下面分别举一些例子进行说明。

- #### 条件注解

    - `@ConditionalOnBean`
    
        > 当容器里有指定的bean类或者名称时，满足匹配条件
        
    - `@ConditionalOnMissingBean`
    
        > 当容器里缺少指定的bean类或者名称时，满足匹配条件
        
    - `@ConditionalOnProperty`
    
        > 检查指定的属性是否有特定的值，name为指定值的key，havingValue为指定值，matchIfMissing为缺少或为设置值时，是否也进行匹配
        
    - `@ConditionOnClass`
    
        > 当有指定类在classpath时，满足匹配条件
    
    - `@ConditionOnMissClass`
    
        > 当指定类在classpath不存在时，满足匹配条件
    

- #### 配置相关注解

    - `@EnableAutoConfiguration`
    
        > 开启自动配置

    - `@AutoConfigureAfter`
    
        > 在指定的自动配置类之后进行配置
        
    - `@AutoConfigureBefore`
    
        > 在指定的自动配置类之前进行配置
    
    - `@EnableConfigurationProperties`
    
        > 开启对`@ConfigurationProperties`修饰的bean的支持，将`@ConfigurationProperties`修饰的类注册成bean
 
---

&emsp;&emsp;而最核心的我想就是`@EnableAutoConfiguration`注解了，毕竟只有有了它才能开启自动配置，是所有自动配置的起点，那么就从它开始探索下自动配置的原理。

> 以下源码版本为springboot 2.0.5.RELEASE

先看下这个注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```

&emsp;&emsp;可以看到这里导入了`AutoConfigurationImportSelector`，先看下该类核心的`selectImports()`方法：

```java
@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}
```

&emsp;&emsp;可以看出，先获取到所有配置，然后删除重复的配置，然后获取需要排除的配置类，再检查需要排除的配置类，从所有配置中移除需要排除的配置，再使用filter过滤掉不满足自动导入条件的配置（例如使用了`@Contional`及衍生的条件注解的配置类），最后将剩下的配置类返回。

&emsp;&emsp;下面我们具体地看下`selectImports()`中的`getCandidateConfigurations()`:

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

&emsp;&emsp;该方法主要通过`SpringFactoriesLoader.loadFactorynames()`使用当前的类加载器返回指定`EnableAutoConfiguration`在`META-INF/spring.factories`对应的所有配置类名。

&emsp;&emsp;那么`META-INF/spring.factories`是什么呢，在spring-boot-autoconfigure.jar包下有这样一个文件：

```xml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
...省略...

```

&emsp;&emsp;可以看到，这里几乎写了所有springboot提供的各种配置类，所以`@EnableAutoConfiguration`导入了`AutoConfigurationImportSelector`，就几乎相当于将所有的配置类都引入进来加载了，所以springboot的自动配置换句话说，其实是帮我们定义了很多通用配置的配置类，再统一按条件引入进来加载。

&emsp;&emsp;至于`selectImports()`中其他作用的实现都比较简单，值得一看的是通过`filter(configurations, autoConfigurationMetadata)`过滤不满足条件的配置类，源码：

```java
private List<String> filter(List<String> configurations,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		long startTime = System.nanoTime();
		String[] candidates = StringUtils.toStringArray(configurations);
		boolean[] skip = new boolean[candidates.length];
		boolean skipped = false;
		for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
			invokeAwareMethods(filter);
			boolean[] match = filter.match(candidates, autoConfigurationMetadata);
			for (int i = 0; i < match.length; i++) {
				if (!match[i]) {
					skip[i] = true;
					skipped = true;
				}
			}
		}
		if (!skipped) {
			return configurations;
		}
		List<String> result = new ArrayList<>(candidates.length);
		for (int i = 0; i < candidates.length; i++) {
			if (!skip[i]) {
				result.add(candidates[i]);
			}
		}
		if (logger.isTraceEnabled()) {
			int numberFiltered = configurations.size() - result.size();
			logger.trace("Filtered " + numberFiltered + " auto configuration class in "
					+ TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
					+ " ms");
		}
		return new ArrayList<>(result);
	}
```

&emsp;&emsp;这里主要是从`META-INF/spring.factories`中获取到`AutoConfigurationImportFilter`：

```xml
# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnClassCondition
```

&emsp;&emsp;即`OnClassCondition`，调用`match()`方法检查是否存在有`@ConditionalOnClass`和`@ContionalOnMissClass`注解的配置类。

&emsp;&emsp;至此，关于`@EnableAutoConfiguration`的自动配置原理我们心里也有了答案。

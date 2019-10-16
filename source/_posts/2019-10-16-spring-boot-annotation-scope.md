---
title: SpringBoot：@Scope注解学习
description: springboot的@Scope注解新理解
categories:
 - 后端
 - Spring
tags:
 - Java
 - spring
---


&emsp;&emsp;最近在项目中遇到了一个bug，在解决bug的过程中，发现自己以往对于`@Scope`注解的理解一直存在问题，甚至是有些错误的，通过排查项目中bug以及写demo、查阅资料，对于`@Scope`有了新的认识。

&emsp;&emsp;先通过注解的javadoc，可以了解到，`@Scope`在和`@Component`注解一起修饰在类上，作为类级别注解时，`@Scope`表示该类实例的范围，在和`@Bean`一起修饰在方法上，作为方法级别注解时，`@Scope`表示该方法返回的实例的范围。

&emsp;&emsp;对于`@Scope`注解，我们常用的属性一般就是：`value`和`proxyMode`，`value`就是指明使用哪种作用域范围，`proxyMode`指明使用哪种作用域代理。

&emsp;&emsp;`@Scope`定义提供了的作用域范围一般有：`singleton`单例、`prototype`原型、`request`web请求、`session`web会话，同时我们也可以自定义作用域。

- 作用域范围

&emsp;&emsp;**`singleton`单例范围**，这个是比较常见的，Spring中bean的实例默认都是单例的，单例的bean在Spring容器初始化时就被直接创建，不需要通过`proxyMode`指定作用域代理类型。

&emsp;&emsp;**`prototype`原型范围**，这个使用较少，这种作用域的bean，每次注入调用，Spring都会创建返回不同的实例，但是，需要注意的是，如果未指明代理类型，即不使用代理的情况下，将会在容器启动时创建bean，那么每次并不会返回不同的实例，只有在指明作用域代理类型例如`TARGET_CLASS`后，才会在注入调用每次创建不同的实例。

&emsp;&emsp;**`request`web请求范围**，（最近遇到的问题就是和`request`作用域的bean有关，才发现之前的理解有偏差），当使用该作用域范围时（包括下面的`session`作用域），必须指定`proxyMode`作用域代理类型，否则将会报错，对于`request`作用域的bean，（<del>之前一直理解的是每次有http请求时都会创建</del>），但实际上并不是这样，而是**Spring容器将会创建一个代理用作依赖注入，只有在请求时并且请求的处理中需要调用到它，才会实例化该目标bean**。

&emsp;&emsp;**`session`web会话范围**，这个和`request`类似，同样必须指定`proxyMode`，而且也是**Spring容器创建一个代理用作依赖注入，当有会话创建时，并且在会话中请求的处理中需要调用它，才会实例话该目标bean，由于是会话范围，生命依赖于session**。


- 作用域代理

&emsp;&emsp;如果指定为`proxyMode = ScopedProxyMode.TARGET_CLASS`，那么将使用cglib代理创建代理实例；如果指定为`proxyMode = ScopedProxyMode.INTERFACE`，那么将使用jdk代理创建代理实例；如果不指定，则直接在Spring容器启动时创建该实例。而且**使用代理创建代理实例时，只有在注入调用时，才会真正创建类对象**。


---

&emsp;&emsp;除了上述作用域范围，Spring也允许我们自定义范围，主要操作为：

- 先实现`Scope`接口创建自定义作用域范围类
- 使用`CustomScopeConfigurer`注册自定义的作用域范围


&emsp;&emsp;后面写了一个例子实践一下，自定义了一种同一分钟的作用域范围，即同一分钟获取的是相同实例。

&emsp;&emsp;首先自定义作用域范围类`TimeScope`:

```java
@Slf4j
public class TimeScope implements Scope {

    private static Map<String, Map<Integer, Object>> scopeBeanMap = new HashMap<>();

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Integer hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY);
        // 当前是一天内的第多少分钟
        Integer minute = hour * 60 + Calendar.getInstance().get(Calendar.MINUTE);
        log.info("当前是第 {} 分钟", minute);
        Map<Integer, Object> objectMap = scopeBeanMap.get(name);
        Object object = null;
        if (Objects.isNull(objectMap)) {
            objectMap = new HashMap<>();
            object = objectFactory.getObject();
            objectMap.put(minute, object);
            scopeBeanMap.put(name, objectMap);
        } else {
            object = objectMap.get(minute);
            if (Objects.isNull(object)) {
                object = objectFactory.getObject();
                objectMap.put(minute, object);
                scopeBeanMap.put(name, objectMap);
            }
        }
        return object;
    }

    @Override
    public Object remove(String name) {
        return scopeBeanMap.remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
    }
    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }
    @Override
    public String getConversationId() {
        return null;
    }
}
```

> `Scope`接口提供了五个方法，只有`get()`和`remove()`是必须实现，`get()`中写获取逻辑，如果已有存储中没有该名称的bean，则通过`objectFactory.getObject()`创建实例。

&emsp;&emsp;然后注册自定义的作用域范围：

```java
@Configuration
@Slf4j
public class BeanScopeConfig {
    @Bean
    public CustomScopeConfigurer customScopeConfigurer() {
        CustomScopeConfigurer customScopeConfigurer = new CustomScopeConfigurer();
        Map<String, Object> map = new HashMap<>();
        map.put("timeScope", new TimeScope());
        customScopeConfigurer.setScopes(map);
        return customScopeConfigurer;
    }
    
    @Bean
    @Scope(value = "timeScope", proxyMode = ScopedProxyMode.TARGET_CLASS)
    public TimeScopeBean timeScopeBean() {
        TimeScopeBean timeScopeBean = new TimeScopeBean();
        timeScopeBean.setCurrentTime(System.currentTimeMillis());
        log.info("time scope bean");
        return timeScopeBean;
    }
}
```

&emsp;&emsp;然后注入调用`timeScopeBean`，同一分钟内重复调用，使用相同实例，不同分钟将创建新实例。


---


&emsp;&emsp;关于`@Scope`新的理解认识就这些了，以后再有新理解再补充吧 (⊙﹏⊙)b
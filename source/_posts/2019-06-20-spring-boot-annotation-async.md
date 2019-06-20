---
title: SpringBoot：@Async注解详解
description: 探索springboot的@Async注解
categories:
 - 后端
 - Spring
tags:
 - Java
 - spring
---



在springboot中，提供了`@Async`注解来实现异步调用，方法使用`@Async`修饰之后，就变成了异步方法，这些方法将会在单独的线程被执行，方法的调用者不用在等待方法的完成。

#### `@Async`的启用与使用

 - 在配置类或者启动类上加上`@EnableAsync`注解

例如下面代码：


```java
@SpringBootApplication
@EnableAsync
public class StudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(StudyApplication.class, args);
    }
}
```

 - 无返回值的方法使用`@Async`

在service上方法上添加`@Async`注解：

```java
@Service
@Slf4j
public class AsyncService {

    @Async
    public void test() {
        log.info("test");
    }
}
```

再在controller中调用service该方法：

```java
@RestController
public class AsyncController {

    @Autowired
    AsyncService asyncService;

    @GetMapping("/async/test2")
    public void testAsync() {
        asyncService.test();
    }
}
```

 - 有返回值的方法使用`@Async`

springboot提供了`AsyncResult`类来支持`@Async`处理，该类实现了`ListenableFuture`接口，而该接口则继承自`Future`接口，所以对于需要有返回值的方法，可将返回值用Future包装，如下：

```java
    @Async
    public Future<String> testFuture() {
        log.info("test future:");
        try {
            Thread.sleep(5000L);
            return new AsyncResult<String>("test");
        } catch (InterruptedException e) {
            e.printStackTrace();
            return null;
        }
    }
```

> controller里通过future.get()获取返回值，但需要注意的是，该方法会阻塞


#### `@Async`源码探索

我们都知道，被`@Async`修饰的方法都会被单独的线程执行，而spring容器需要维护这些线程，就需要一个线程池，而默认提供的线程池就是`SimpleAsyncTaskExecutor`，我们通过打印当前线程名就能看出：

```java
test:Thread[SimpleAsyncTaskExecutor-1,5,main]
```

那么默认的线程池是怎么样被提供的呢，探索一下源码，这里springboot的版本是2.0.5。

首先想要使用`@Async`则需要开启这个功能，即`@EnbaleAsync`注解，我们看一下这个注解源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {}
```

单从源码上，可以看到引入了一个selector：`AsyncConfigurationSelector`，我们再看这个注解的注释，可以发现些端倪：

```java
* <p>By default, Spring will be searching for an associated thread pool definition:
 * either a unique {@link org.springframework.core.task.TaskExecutor} bean in the context,
 * or an {@link java.util.concurrent.Executor} bean named "taskExecutor" otherwise. If
 * neither of the two is resolvable, a {@link org.springframework.core.task.SimpleAsyncTaskExecutor}
 * will be used to process async method invocations. Besides, annotated methods having a
 * {@code void} return type cannot transmit any exception back to the caller. By default,
 * such uncaught exceptions are only logged.
```

从注释中我们不难发现，默认情况，会去spring上下文中找`TaskExecutor`、`Executor`的bean或者名字是taskExecutor的bean，如果都没有，则会默认使用`SimpleAsyncTaskExecutor`。从注释中我们已经可以确定了，但Spring是怎么实现的呢，`@EnableAsync`中引入了`AsyncConfigurationSelector`，我们点开这个类，寻找下是否有实现：

```java
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {

	private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
			"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";
	@Override
	@Nullable
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {ProxyAsyncConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME};
			default:
				return null;
		}
	}

}
```

可以看到，当是jdk代理模式时，使用`ProxyAsyncConfiguration`配置类，当使用aspectJ代理时，使用`org.springframework.scheduling.aspectj.AspectJAsyncConfiguration`，而默认则是jdk代理，所以继续点开`ProxyAsyncConfiguration`：

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {

	@Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
		Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
		AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
		Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
		if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
			bpp.setAsyncAnnotationType(customAsyncAnnotation);
		}
		if (this.executor != null) {
			bpp.setExecutor(this.executor);
		}
		if (this.exceptionHandler != null) {
			bpp.setExceptionHandler(this.exceptionHandler);
		}
		bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
		bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
		return bpp;
	}

}
```

可以看到熟悉的字眼：executor，而这里的executor来自父类`AbstractAsyncConfiguration`，难道就要揭开谜底了吗，怀着激动的心情我们继续看这个父类：

```java
@Configuration
public abstract class AbstractAsyncConfiguration implements ImportAware {

	// 省略

	@Nullable
	protected Executor executor;

	// 省略


	@Override
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		// 省略
	}

	@Autowired(required = false)
	void setConfigurers(Collection<AsyncConfigurer> configurers) {
		if (CollectionUtils.isEmpty(configurers)) {
			return;
		}
		if (configurers.size() > 1) {
			throw new IllegalStateException("Only one AsyncConfigurer may exist");
		}
		AsyncConfigurer configurer = configurers.iterator().next();
		this.executor = configurer.getAsyncExecutor();
		this.exceptionHandler = configurer.getAsyncUncaughtExceptionHandler();
	}

}
```

不禁哑然，executor是通过`setConfigurers()`方法赋值的，而该方法注入了`AsyncConfigurer`类型的bean，通过`configurer.getAsyncExecutor()`获取到executor，而可惜的是`AsyncConfigurer`接口默认返回null，而它唯一的子类，也是返回null，也就是说整个spring上下文并没有该类型的bean，即executor为null，但上面我们明明通过打印日志和注释知道了默认是`SimpleAsyncTaskExecutor`。

> 其实这里也指明了一种我们提供自定义线程池的方式，定义一个类实现`AsyncConfigurer`接口，实现`getAsyncExecutor`方法返回自定义的executor，再将该类注册成bean。

那么就很明显了，`@EnableAsync`这条线索是走不通了，默认线程池并不是在开启async的时候提供的，那么就有可能和`@Async`注解相关。

`@Async`这个注解的实现是基于spring-aop来做到的，注解源码本身并没什么，看注释：

```java
 * @see AnnotationAsyncExecutionInterceptor
 * @see AsyncAnnotationAdvisor
```

`AnnotationAsyncExecutionInterceptor`顾名思义，是`@Async`注解修饰的方法执行的拦截器，该类继承自`AsyncExecutionInterceptor`，类本身没什么核心方法，继续看父类`AsyncExecutionInterceptor`中的核心方法`invoke()`：

```java
@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
		final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

		AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
		if (executor == null) {
			throw new IllegalStateException(
					"No executor specified and no default executor set on AsyncExecutionInterceptor either");
		}

		Callable<Object> task = () -> {
			try {
				Object result = invocation.proceed();
				if (result instanceof Future) {
					return ((Future<?>) result).get();
				}
			}
			catch (ExecutionException ex) {
				handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
			}
			catch (Throwable ex) {
				handleError(ex, userDeclaredMethod, invocation.getArguments());
			}
			return null;
		};

		return doSubmit(task, executor, invocation.getMethod().getReturnType());
	}
```

可以看到有这么一行，确定了async使用的executor：

```java
AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
```

该方法来自`AsyncExecutionInterceptor`的父类`AsyncExecutionAspectSupport`，看一下该方法：

```java
protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
		AsyncTaskExecutor executor = this.executors.get(method);
		if (executor == null) {
			Executor targetExecutor;
			String qualifier = getExecutorQualifier(method);
			if (StringUtils.hasLength(qualifier)) {
				targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
			}
			else {
				targetExecutor = this.defaultExecutor;
				if (targetExecutor == null) {
					synchronized (this.executors) {
						if (this.defaultExecutor == null) {
							this.defaultExecutor = getDefaultExecutor(this.beanFactory);
						}
						targetExecutor = this.defaultExecutor;
					}
				}
			}
			if (targetExecutor == null) {
				return null;
			}
			executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
					(AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
			this.executors.put(method, executor);
		}
		return executor;
	}
```

这里this.executor初始时是一个空的Map，初始时this.defaultExecutor也是为空，所以将会执行其中的`this.defaultExecutor = getDefaultExecutor(this.beanFactory);`，而子类`AsyncExecutionInterceptor`重写了该方法，再回到`AsyncExecutionInterceptor`查看：

```java
@Override
	@Nullable
	protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
		Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
		return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
	}
```

看到这里似乎有拨开云雾见青天的意思了，会先调用父类`AsyncExecutionAspectSupport`的`getDefaultExecutor()`方法，如果不为空，则返回父类获取的executor，如果为空，则返回`SimpleAsyncTaskExecutor`，而父类`AsyncExecutionAspectSupport`的`getDefaultExecutor()`：

```java
@Nullable
	protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
		if (beanFactory != null) {
			try {
			    beanFactory.getBean(TaskExecutor.class);
			}
			catch (NoUniqueBeanDefinitionException ex) {
				logger.debug("Could not find unique TaskExecutor bean", ex);
				try {
					return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
				}
				catch (NoSuchBeanDefinitionException ex2) {
					if (logger.isInfoEnabled()) {
						logger.info("More than one TaskExecutor bean found within the context, and none is named " +
								"'taskExecutor'. Mark one of them as primary or name it 'taskExecutor' (possibly " +
								"as an alias) in order to use it for async processing: " + ex.getBeanNamesFound());
					}
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				logger.debug("Could not find default TaskExecutor bean", ex);
				try {
					return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
				}
				catch (NoSuchBeanDefinitionException ex2) {
					logger.info("No task executor bean found for async processing: " +
							"no bean of type TaskExecutor and no bean named 'taskExecutor' either");
				}
			}
		}
		return null;
	}
```

这里其实就是从Spring上下文获取TaskExecutor类型的bean，或者名称为taskExecutor的bean。

而获取到executor之后，会将被@Async修饰的方法，也就是任务task，提交（submit）到excutor中执行。

所以到这里，关于`@Async`默认线程池的使用有了答案，默认会使用`SimpleAsyncTaskExecutor`，而对于我们，想要自定义executor时，有两种方法，一种就是上面说的，自定义类实现`AsyncConfigurer`接口再注册成bean，另一种就是注册TaskExecutor类型的bean或者名称是taskExecutor的bean。

> 以上探索的过程主要通过查看源码以及源码注释，以及结合debug模式下查看调用栈



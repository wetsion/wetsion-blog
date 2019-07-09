---
title: Mybatis源码学习：mapper关于事务的补充
description: Mybatis中关于事务的学习
categories:
 - 后端
 - Mybatis
tags:
 - Java
 - Mybatis
---


&emsp;&emsp;最近在整理某个系统需要优化的点的时候，发现在查询时速度很慢，经过排查代码，发现同事在某个service出现了循环调用另一个service的方法，而另一份servcei是查询数据库的，而这些service都没使用`@@Transactional`事务，同时，根据debug打印的sql查询语句，发现单条sql语句查询都很快，初步怀疑是频繁创建会话，重复打开关闭连接消耗了大量时间，于是重写了这个service的方法，直接使用相应的mapper，然后使用`@@Transactional`声明为一个事务，这样所有的sql都在一个会话中了，只打开关闭了一次连接，响应的速度极大的提高了。

&emsp;&emsp;那么，在之前的mapper注册注入的文章中，学习到通过注入mapper的方式时，每次调用方法都是创建一个sqlsession。而使用`@@Transactional`事务注解后，service方法中所有mapper的调用都是一个会话了，在service方法结束之后统一对会话进行提交，那么这又是怎么实现的呢。

&emsp;&emsp;通过之前的学习，已知道注入后，本质上是`SqlSessionTemplate`通过内部代理类`SqlSessionInterceptor`对象`sqlSessionProxy`，再看一遍`SqlSessionInterceptor`的`invoke()`方法：

```java
@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
```

&emsp;&emsp;先`getSqlSession()`：

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }
    LOGGER.debug(() -> "Creating a new SqlSession");
    session = sessionFactory.openSession(executorType);
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
    return session;
  }
```

&emsp;&emsp;从`getSqlSession()`这个方法中可以看到，先通过`TransactionSynchronizationManager.getResource(sessionFactory)`获取`SqlSessionHolder`，从`TransactionSynchronizationManager`类的名字似乎是关于事务的，进入getResource这个方法：

```java
    private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");
    @Nullable
	public static Object getResource(Object key) {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
		Object value = doGetResource(actualKey);
		// 日志打印
		return value;
	}
	private static Object doGetResource(Object actualKey) {
		Map<Object, Object> map = resources.get();
		if (map == null) {
			return null;
		}
		Object value = map.get(actualKey);
		if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
			map.remove(actualKey);
			if (map.isEmpty()) {
				resources.remove();
			}
			value = null;
		}
		return value;
	}
```

&emsp;&emsp;可以看到，该类的`resources`属性是一个ThreadLocal对象，泛型是`Map<Object, Object>`，也就是说保存和当前线程绑定的一个map，然后根据key获取map中保存的对象

> service层方法中mapper的调用都是在一个线程中进行的

&emsp;&emsp;而`SqlSessionUtils#getSqlSession()`根据sessionFactory获取了一个SqlSessionHolder对象，反推之，说明在某个过程中，曾经放入了一个SqlSessionHolder对象，其实在这个方法后面有句`registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);`这就是放入的操作，这个操作是在获取不到`SqlSessionHolder`，也就获取不到session之后，创建一个新的sqlSession，然后构造一个`SqlSessionHolder`对象，注册到`TransactionSynchronizationManager`的`resources`map中，具体看下`registerSessionHolder()`这个方法：


```java
private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
    SqlSessionHolder holder;
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      Environment environment = sessionFactory.getConfiguration().getEnvironment();
      if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
        LOGGER.debug(() -> "Registering transaction synchronization for SqlSession [" + session + "]");
        holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
        TransactionSynchronizationManager.bindResource(sessionFactory, holder);
        TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
        holder.setSynchronizedWithTransaction(true);
        holder.requested();
      } else {
        if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
          LOGGER.debug(() -> "SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
        } else {
          throw new TransientDataAccessResourceException(
              "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
        }
      }
    } else {
      LOGGER.debug(() -> "SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
    }
 }
```

&emsp;&emsp;通过`TransactionSynchronizationManager.isSynchronizationActive()`判断有事务之后（每当开启一个事务，就会调用`TransactionSynchronizationManager#initSynchronization()`对synchronizations初始化），并且判断环境中有spring管理的`TransactionFactory`bean之后，构造SqlSessionHolder对象与sessionFactory绑定，注册到`TransactionSynchronizationManager`的`resources`map中。注意，这里有个操作`TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));`，将在后面说到，在service层方法执行完，统一提交事务时会涉及到。

&emsp;&emsp;结合上述，回到`SqlSessionInterceptor#invoke()`，可以知道，会尝试从`TransactionSynchronizationManager`的resources获取当前线程当前sessionFactory对应的SqlSessionHolder，如果之前放入过，则直接取出SqlSessionHolder中持有的sqlSession，如果没，则创建sqlSession返回，再根据是否有声明事务，构造SqlSessionHolder持有sqlSession，再注册进`TransactionSynchronizationManager`的resources，返回sqlSession。

&emsp;&emsp;拿到sqlSession后，通过`method.invoke(sqlSession, args)`直接调用`SqlSession`的数据操作，获取结果。然后通过`isSqlSessionTransactional()`判断是否具有事务（尝试从`TransactionSynchronizationManager`的resources中获取当前线程对应的SqlSessionHolder，如果为空，或者持有的sqlSession不等于当前的sqlSession，说明不具有事务），如果无事务，那么就`sqlSession.commit(true)`立即提交当前会话。在返回结果之前，会执行`closeSqlSession()`（ 即同上再次尝试从resources获取`SqlSessionHolder`，如果不具有事务，则关闭会话）。


> `sqlSession.commit()`包含了提交事务的作用，但在与spring的事务整合时发现，一些属性会影响使其执行不到其中`transaction.commit()`，然后靠后续通过`Connection`调用`NaticeSession`发送`commit`指令给mysql来提交

&emsp;&emsp;从以上了解到了没有事务情况下，每次mapper调用都会创建新的会话，并且会话会在执行完后立即提交，而有事务时，共用一个会话，执行完数据操作后并不会提交会话，通过debug日志知道，在service方法执行完后，统一提交了事务，那么是如何统一提交事务呢，以及为什么未使用事务mapper执行完就提交会话了呢。

&emsp;&emsp;service的方法在被执行调用时，实际上是被代理类`CglibAopProxy`动态代理，并进入到通用回调`DynamicAdvisedInterceptor`中，看一下这个回调中的`intercept()`:

```java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
				targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
				AopContext.setCurrentProxy(oldProxy);
				}
			}
		}
```

&emsp;&emsp;这里是一个关键点，对于有事务和无事务的分岔点就在这里，检查该方法（切点）的`InvokerInterceptor`数量，如果为空，那么就跳过创建`MethodInvocation`，而直接调用代理目标也就是service方法，如果不为空，则创建一个`CglibMethodInvocation`并执行。而如果方法有`@Transactional`注解修饰，该方法（切点）会拥有一个`TransactionInterceptor`，所以就不直接调用service方法，而是创建`CglibMethodInvocation`调用。

#### 无事务的情况

&emsp;&emsp;无事务时，直接调用代理目标也就是service方法，service调用mapper的方法，而调用mapper的方法，从前面的学习已经知道实质上调用`SimpleExecutor`的`doUpdate()`或`doQuery()`，而这两个方法实际上是调用`StatementHandler`的update和query，而再往后则是调用`PreparedStatement`的`execute()`，最终调用mysql的jdbc驱动包中原始会话`NaticeSession`的`execSQL()`，第一次先建立连接，第二次发送`SET autocommit=1`，告诉mysql后续的操作都是自动提交，所以后续直接执行sql获取结果

#### 有事务的情况

&emsp;&emsp;就如上面所说，创建`CglibMethodInvocation`，然后反射调用`TransactionInterceptor`，`TransactionInterceptor`继承自`TransactionAspectSupport`，`TransactionInterceptor`的`invoke()`调用`TransactionAspectSupport`的`invokeWithinTransaction()`：

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {	completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			// CallbackPreferringPlatformTransactionManager情况下的处理
		}
	}
```

&emsp;&emsp;可以观察到，这是一个环绕切面，先通过`retVal = invocation.proceedWithInvocation()`调用代理目标，也就是调用service方法执行切点，然后后置地调用`commitTransactionAfterReturning(txInfo)`，这里是在切点执行之后，用于提交事务的。在调用service方法时，调用mapper的方法，在最终调用mysql驱动包时，建立连接后，给mysql发送了`SET autocommit=0`，即意味着后续的sql执行都先不提交，而在`commitTransactionAfterReturning()`：

```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			// 日志打印省略
			txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
		}
	}
```

&emsp;&emsp;这里获取到的TransactionManager是`DataSourceTransactionManager`，`DataSourceTransactionManager`继承` AbstractPlatformTransactionManager`，commit方法中再次调用`processCommit()`，`processCommit()`调用`doCommit()`，在`doCommit()`中获取连接`Connection`，调用`Connection#commit()`，最终也是调用`NativeSession#execSQL()`，只不过给mysql发送的是`commit`，即提交之前发送的所有sql。

> 在`processCommit()`中调用`doCommit()`之前，有一步调用`triggerBeforeCommit(status)`，而在这个方法中调用`TransactionSynchronizationUtils.triggerBeforeCommit(status.isReadOnly())`，而在前文中，提到过，在创建sqlSession时，会构造SqlSessionHolder，然后注册到`TransactionSynchronizationManager`，同时`TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));`创建了`SqlSessionSynchronization`对象注册。而`TransactionSynchronizationUtils.triggerBeforeCommit(status.isReadOnly())`中即对`TransactionSynchronizationManager`中所有的`TransactionSynchronization`对象遍历，执行它们的`beforeCommit()`，而`SqlSessionSynchronization`的`beforeCommit()`中，获取持有sqlSessionHolder进而获取SqlSession，然后调用`commit()`。


学习到这里，对mybatis的会话关于事务方面做了些补充，但学习的越多，疑惑也就越多（坑也就越多...），比如关于Spring如何管理事务的，将带着这些疑问进行后续的学习，做好学习笔记。

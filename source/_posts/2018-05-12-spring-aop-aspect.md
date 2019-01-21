---
title: Spring与SpringBoot：SpringAOP使用@Aspect
description: spring aop 中使用@Aspect注解
categories:
 - 后端
 - Spring
tags:
 - spring
 - aop
---
&emsp;&emsp;在spring中，AOP和IOC都是spring非常重要的特性，而在web开发中，定义切面、增强方法也是比较常见的，比如做统一的日志管理相关的、自定义的注解处理、或者在处理用户请求的前后我们需要做一些处理，等等，这时我们都可以使用切面来实现，而在以前，使用切面我们可能需要使用很多接口和类，现在，我们只需要@Aspect这一个注解就可以定义切面。 	

&emsp;&emsp;首先，我们定义一个类，然后在这个类上使用@Aspect注解，这样一个切面就定义好了，我们可以再加上@Component注解，这样就把这个切面交给Spring容器来管理。

```java
@Aspect
@Component
public class RecommendLogAspect {
}
```

接下来就是定义切点，并对切点做一些增强操作：前置增强、环绕增强、后置增强等等，切点的定义我们可以在一个空方法体的方法上使用@Pointcut注解

```java
@Pointcut("@annotation(com.sino.baseline.common.annotation.InitQueryParam)")
	public void pointcut() {}
```

@Pointcut()里面定义的是切点表达式，切点表达式有很多，上面例子代码中的是注解表达式，标注来指定注解的目标类或者方法，就比如凡是使用了com.sino.baseline.common.annotation.InitQueryParam这个注解的类或者方法都是切点。除了@annotation()还有几类比较常见的切点表达式：

1.execution(方法修饰符 返回类型 方法全限定名 参数)  
匹配指定的方法，例如.  

```Java
@Pointcut("execution(* com.tcb.controller.SDProductController.showproductDetail(..))")
```  
>  \* 匹配任意字符，但只能匹配一个元素  
>  .. 匹配任意字符，可以匹配任意多个元素，表示类时，必须和*联合使用  
>  \+ 必须跟在类名后面，如Student+，表示类本身和继承或扩展指定类的所有类  

2.args( 参数类型的类全限定名 )  
匹配参数是指定类型的方法，比如@Pointcut(com.xx.Student) 就是匹配所有参数是student类对象的方法，像void add(Student s)这样的方法就会被匹配。

3.@args( 注解类的类全限定名 )  
匹配参数的类型的类被指定注解修饰的方法，注意，这个参数的类型还必须是自定义的类，比如@Pointcut(com.xx.anno.ExAnno)，你有一个方法void(Student s){}，而参数类型Student是你自定义的类，而Student这个类被@ExAnno修饰了，那么，这个方法就会被匹配（这个地方有点坑！）。  
4.within( 类全限定名 )  
匹配指定的类，比如@Pointcut(com.xx.IndexController)就是IndexController类下的方法都会被匹配，这里的类名支持正则模糊匹配，比如@Pointcut(com.xx.*Controller)就是com.xx包下的所有的Controller都会被匹配。对了，上面和下面的表达式都支持正则模糊匹配  
5.target( 类全限定名 )  
匹配指定的类以及它的子类，比如@Pointcut(com.xxx.dao.BaseDao)就是匹配BaseDao接口以及所有实现类这个接口的子类。  
6.@within( 类全限定名 )  
匹配使用了指定的注解的类以及它的子类，比如@Pointcut(com.xxx.anno.Log)就是指，我在BaseController里使用了@Log注解，那么BaseController以及继承BaseController的类都会被匹配。  
7.@target( 类全限定名 )  
匹配使用了指定注解的类（从这里我们看出来，within、target和@within、@target是相反，郁闷。。。），还是@Pointcut(com.xxx.anno.Log)就是仅匹配使用了@Log注解的类

--

ok，以上是一些较常用的切点表达式，然后继续之前的。  
在定义切点之后，我们就可以对切点进行增强操作（注意，我前面@Pointcut注解修饰的方法名是pointcut()哦），比如@Before("pointcut()")前置增强，@Around("pointcut()")环绕增强。。。等等，注意，我这里直接使用的前面定义切点的那个方法名，这种方式属于独立切点命名（划重点！），就是将切点单独定义出来，对于当切点较多时，能够提高一些重用性。这种算是显式地定义切点，除了这种还可以使用匿名切点，即直接在增强操作方法里直接写切点表达式，比如:  
 
```java
@Before("execution(* com.tcb.controller.SDProductController.showproductDetail(..))")
	public void beforeBrowse(JoinPoint joinPoint) {}
```

好了，增强主要有以下几种：  
1.@Before  
前置增强，在切点方法执行之前执行。这里多说几句，在增强方法中要获取request可以通过下面来获取：  

```java
ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
HttpServletRequest request = attributes.getRequest();
```

在前置增强中，想要获取切点方法的参数可以通过joinPoint.getArgs[]来获取，获取方法名可以通过joinPoint.getSignature().getDeclaringTypeName()来获取。  
2.@Around  
环绕增强.  
3.@AfterReturning  
后置增强，切点方法正常执行完返回后执行，如果有异常抛出而退出，则不会执行增强方法   
4.@AfterThrowing  
后置增强，只有切点方法异常抛出而退出后执行  
5.@After  
也是后置增强，但不管切点方法是正常退出还是异常退出都会执行

好了，以上就是AOP开发基于@Aspect等注解的使用方法，有新的心得再随时修改或者写第二篇，有不对的地方欢迎指摘，吃饭喽
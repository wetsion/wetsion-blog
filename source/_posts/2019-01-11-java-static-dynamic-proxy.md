---
title: Java中的静态代理和动态代理
description: Java静态代理与动态代理的实现
categories:
 - 后端
 - Java
tags:
 - Java
---


在学习SpringAOP之前，首先需要先对Java的静态代理和动态代理进行了解

## 静态代理

所谓静态代理，就是让代理对象和目标对象实现共同的接口，并且代理对象中包含目标对象的引用


可以通过一个例子来理解静态代理，首先我们定义抽象的动物接口`IAnimal` :

```java
public interface IAnimal {
    void say();
}
```

接着定义一个目标对象的类`Dog`，实现了`IAnimal`接口： 

```java
public class Dog implements IAnimal {
    @Override
    public void say() {
        System.out.println("汪汪汪");
    }
}
```  
然后定义代理对象的类`ProxyDog`，在代理对象中定义目标的引用：  
```java
public class ProxyDog implements IAnimal {

    private IAnimal dog;

    public ProxyDog(IAnimal dog) {
        super();
        this.dog = dog;
    }

    @Override
    public void say() {
        dog.say();
    }
}
```  
最后看下该怎么使用代理对象：  
```java
public class StaticTest {
    public static void main(String[] args) {
        IAnimal proxyDog = new ProxyDog(new Dog());
        proxyDog.say();
    }
}
```

可以看到，静态代理本质上是在代理对象内部持有目标对象的引用，由于和目标对象实现了一样的接口，所以调用方法一样，在同样的调用方法里去调用目标对象的该方法，从而达到代理的目的。

## 动态代理

动态代理通过实现`java.lang.reflect.InvocationHandler`接口，将目标对象引入进来，然后利用反射机制执行目标对象的方法。

还是上面的`IAnimal`接口，新写一个`Cat`实现类：  
```java
public class Cat implements IAnimal {
    @Override
    public void say() {
        System.out.println("喵喵喵");
    }
}
```

然后定义个实现了`InvocationHandler`的代理类`DynamicCat`：  
```java
public class DynamicCat implements InvocationHandler {

    private IAnimal cat;

    public IAnimal builder(IAnimal cat) {
        this.cat = cat;
        return (IAnimal) Proxy.newProxyInstance(cat.getClass().getClassLoader(),
                this.cat.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return method.invoke(this.cat, args);
    }
}
```

通过`#builder()`方法将目标对象引入，并通过`Proxy.newProxyInstance`创建代理对象，当调用方法时，本质上调用invoke方法，执行目标对象的方法：

```java
public class DynamicTest {
    public static void main(String[] args) {
        IAnimal proxyCat = new DynamicCat().builder(new Cat());
        proxyCat.say();
    }
}
```
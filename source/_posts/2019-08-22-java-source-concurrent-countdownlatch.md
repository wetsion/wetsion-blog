---
title: Java源码学习：CountDownLatch学习
description: 继续学习基于AQS实现的计数器CountDownLatch
categories:
 - 后端
 - Java
tags:
 - Java
 - 并发
---



&emsp;&emsp;前面相继学习了`ReentrantLock`和`Semaphore`，都是基于AQS来实现的，这次学习的`CountDownLatch`同样也是基于AQS实现的。

&emsp;&emsp;`CountDownLatch`可以理解是一个倒计时计数器，是用于让一个或多个线程等待在其他线程的操作执行完之后再执行的同步辅助。值得注意的是。这里的倒计时计数是一次性的，计数无法重置，即调用`countDown()`方法来计数，`await()`方法等待阻塞直到计数减到零，到达零之后释放所有等待的线程，后续任何的`await()`调用都是立即返回。

&emsp;&emsp;talk is cheap, show me code!还是从一个例子开始入手：

```java
public class CountDownLatchTest {
    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(2);
        new Thread(){
            @Override
            public void run() {
                try {
                    System.out.println("正在攒钱买房...");
                    Thread.sleep((long) (Math.random()*10000));
                    latch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                try {
                    System.out.println("正在攒钱买车...");
                    Thread.sleep((long) (Math.random()*10000));
                    latch.countDown();
                    System.out.println("还有" + latch.getCount() + "个目标没完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
        //主线程
        System.out.println("等待有车有房后就结婚");
        try {
            //等待，阻塞点
            latch.await();
            System.out.println("已经买车买房了，准备结婚了！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：

```java
正在攒钱买房...
正在攒钱买车...
等待有车有房后就结婚
还有1个目标没完成
已经买车买房了，准备结婚了！
```

&emsp;&emsp;栗子就像社会一样现实，想要结婚（执行人生的主线程任务），就得先完成攒钱买车买房（这些子线程任务）。

&emsp;&emsp;首先初始化`CountDownLatch`，传参是2，即定义为两个待完成目标：

```java
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    // Sync
    Sync(int count) {
        setState(count);
    }
```

&emsp;&emsp;构造方法中即初始化内部类`Sync`（同`ReentrantLock`、`Semaphore`一样，都是AQS的实现类），`Sync`的构造方法中调用AQS的`setState()`方法，设置初始同步状态state，也可以理解为需要等待的目标数。

&emsp;&emsp;在执行的子线程中调用`countDown()`方法来倒计时：

```java
    public void countDown() {
        sync.releaseShared(1);
    }
    // AQS
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    // CountDownLatch Sync
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
```

&emsp;&emsp;这里其实类似其他类的`release()`方法，在`Sync`的`tryReleaseShared()`中，自旋，获取同步状态值state，如果state等于0，说明已经倒计时到0了，没有需要等待完成的目标了，返回false；如果不为0，则减一，再通过CAS修改state的值，如果修改成功，再判断减一之后的值是否等于0。这里如果返回true，才会调用AQS的`doReleaseShared()`唤醒在等待的线程。

&emsp;&emsp;那么在等待的线程是如何进入等待的呢，栗子中的结婚主线程是调用`await()`等待阻塞的：

```java
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    // AQS
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
    // CountDownLatch Sync
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
```

&emsp;&emsp;这里倒像是前面学习的那些类的获取锁，通过`getState()`获取同步状态state，如果等于0，则返回1，说明没有需要等待完成的目标了，可以直接执行当前线程；如果不等于0，返回-1，说明还有需要等待完成的目标，则调用AQS的`doAcquireSharedInterruptibly()`方法将当前线程插入到等待队列队尾进行阻塞等待。

> 开头说到`CountDownLatch`计数是一次性的，看到这里也应该有了答案，因为`await()`是需要判断当前同步状态state的值的，而`countDown()`将state不断地减一，并没有其他方法来将state值重置，所以一旦state值到达了0，那么后续其他调用`await()`等待的线程都会直接执行。

&emsp;&emsp;OK，`CountDownLatch`类的实现比较简单，基本上就看完了，类注释中提到，针对需要重置计数，可以考虑使用`CyclicBarrier`，那么下一目标就是学习`CyclicBarrier`了。
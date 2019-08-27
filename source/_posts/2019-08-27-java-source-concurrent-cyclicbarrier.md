---
title: Java源码学习：CyclicBarrier学习
description: 继续学习与CountDownLatch类似的CyclicBarrier
categories:
 - 后端
 - Java
tags:
 - Java
 - 并发
---


&emsp;&emsp;前面刚学了`CountDownLatch`，文末提到了`CyclicBarrier`，可以重置计数，那么这篇笔记就顺便学习下`CyclicBarrier`。

&emsp;&emsp;从`CyclicBarrier`类单词就可以看出来，这是一个循环的屏障、栅栏，让所有线程都被阻止在这个栅栏之前，等到所有线程都到达栅栏（也就是都被阻塞了），然后打开栅栏，让所有被阻止的线程都执行。

&emsp;&emsp;show me code！还是写个栗子：

```java
public class CyclicBarrierTest {
    public static void main(String[] args) {
        final CyclicBarrier cb = new CyclicBarrier(3, new Runnable() {
            @Override
            public void run() {
                System.out.println("铁柱彩礼钱借够了，结婚");
                System.out.println("铁柱老婆带着钱跟隔壁老王跑了");
            }
        });
        ExecutorService threadPool = Executors.newCachedThreadPool();
        for (int i=0;i< 9;i++){
            final int user = i+1;
            Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep((long) (Math.random() * 1000));
                        int k = cb.getNumberWaiting() + 1;
                        System.out.println("铁柱借钱凑彩礼，当前已有" + k + "百万");
                        cb.await();
                        System.out.println("铁柱卖血搬砖还钱");
                    } catch (InterruptedException | BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            };
            threadPool.execute(runnable);
        }
        threadPool.shutdown();
    }
}
```

运行结果：

```java
铁柱借钱凑彩礼，当前已有1百万
铁柱借钱凑彩礼，当前已有2百万
铁柱借钱凑彩礼，当前已有3百万
铁柱彩礼钱借够了，结婚
铁柱老婆带着钱跟隔壁老王跑了
铁柱卖血搬砖还钱
铁柱卖血搬砖还钱
铁柱卖血搬砖还钱
铁柱借钱凑彩礼，当前已有1百万
铁柱借钱凑彩礼，当前已有2百万
铁柱借钱凑彩礼，当前已有3百万
铁柱彩礼钱借够了，结婚
铁柱老婆带着钱跟隔壁老王跑了
铁柱卖血搬砖还钱
铁柱卖血搬砖还钱
铁柱卖血搬砖还钱
铁柱借钱凑彩礼，当前已有1百万
铁柱借钱凑彩礼，当前已有2百万
铁柱借钱凑彩礼，当前已有3百万
铁柱彩礼钱借够了，结婚
铁柱老婆带着钱跟隔壁老王跑了
铁柱卖血搬砖还钱
铁柱卖血搬砖还钱
铁柱卖血搬砖还钱
```

&emsp;&emsp;还是和`CountDownLatch`例子一样现实的例子，为了体现`CyclicBarrier`的可重置，这里假设铁柱借钱凑彩礼钱结婚（在我的老家，借钱凑彩礼钱的现象还是很普遍的），但这次比较倒霉，结婚后老婆卷钱跟隔壁老王跑了，然后忍痛卖血搬砖还钱，然后继续攒钱，最后不断地继续重蹈覆辙。。。

&emsp;&emsp;在这个例子中，「借钱凑彩礼」这类子线程在调用`await()`方法后会被阻塞，直到被阻塞的子线程数量达到了`CyclicBarrier`初始化时定义的3个，那么将会放开，唤醒继续执行这些被阻塞的线程（也就是例子中借钱后的还钱～），如果`CyclicBarrier`使用的是和例子一样的构造方法（参数是阻塞线程数量和栅栏放开触发执行的线程），那么将会先执行定义的栅栏放开触发执行的线程，再继续执行被阻塞的线程。

&emsp;&emsp;从运行结果来看，似乎是当阻塞的线程达到了定义的数量，就会触发栅栏放开，并且栅栏又重置了，那么是怎么实现的呢，还是从学习源码入手。

#### `await()`

```java
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
    
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
    
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
```

&emsp;&emsp;`CyclicBarrier`是基于`ReentrantLock`实现的，内部有一个`ReentrantLock`对象属性，在调用的核心实现`dowait()`方法中，首先使用`lock.lock()`上锁，用来保证`dowait()`操作是同步操作。

&emsp;&emsp;`CyclicBarrier`中有个静态内部类`Generation`，每个`CyclicBarrier`实例中每一回合都会初始化一个`Generation`对象属性: `generation`（每一回合是指栅栏没被破坏或重置，如果被破坏或重置，就是新的一回合），`Generation`类中只有一个属性：`boolean broken`，用于记录当前栅栏是否被破坏（`breakBarrier()`），当栅栏被破坏或重置（`reset()`）时，该值都会改变。

&emsp;&emsp;在`dowait()`中，获取`generation`，判断当前栅栏当前这一回合的broken状态，如果为true，说明被破坏或者重置过，则抛出异常。再判断当前线程是否被中断，如果当前线程被中断，那么这个线程将不可能会到达栅栏，为了避免死锁，调用`breakBarrier()`方法破坏栅栏，并抛出异常。校验过栅栏的broken状态和线程状态之后，将`count`值减一（`count`是`CyclicBarrier`中用于记录栅栏前仍需要等待的线程数量），如果`count`等于0，说明设定数量的线程都已到达了栅栏前，如果在`CyclicBarrier`构造初始化时设置了栅栏放开触发执行的线程，那么执行栅栏放开触发执行的线程（从`command.run()`可以看出，这里执行是同步执行该线程，而非异步并发执行），然后调用`nextGeneration()`方法：

```java
    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```

&emsp;&emsp;调用`nextGeneration()`方法唤醒在条件队列等待的线程（这里是条件队列，trip是`ReentrantLock`的`Condition`对象，调用`signalAll()`方法将条件队列等待的线程添加到同步队列中参与竞争锁，这里参考前面学习AQS关于Condition的笔记，那么条件队列等待的线程是什么时候插入的呢，在后面`await()`将会看到），并重置栅栏前仍需等待的线程数量，以及重新初始化generation对象，**CyclicBarrier的循环重置就是在这里实现的**。

&emsp;&emsp;调用`nextGeneration()`重置之后返回，如果这里出现了异常，则调用`breakBarrier()`破坏栅栏，使失败。

&emsp;&emsp;而对于count不等于0，也就是当前线程到达后，当前线程不是最后一个到达的线程，栅栏前还有线程没到达，则进入循环，直到栅栏被破坏或者线程中断或者超时。在循环中，根据timed参数判断是否是需要超时等待，如果不是，则调用`trip.await()`直接进入等待，即AQS中`ConditionObject#await()`:

```java
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }
```

> 该方法实现可中断的条件等待，`CyclicBarrier`通过调用该方法将当前线程加入到AQS的条件队列中等待。

&emsp;&emsp;如果需要超时等待，并且参数nanos等待时间大于0，则调用`trip.awaitNanos(nanos)`进行超时等待：

```java
    public final long awaitNanos(long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return deadline - System.nanoTime();
    }
```

> 和`await()`差不多，只不过多了超时时间。

&emsp;&emsp;再回到`dowait()`，在加入当前线程到条件队列进行等待之后，判断栅栏当前回合的broken状态，如果已被破坏，则抛出异常；`g != generation`如果已经是新的回合，则直接返回；如果是需要超时等待，但时间已到，则调用`breakBarrier()`破坏栅栏，唤醒在条件队列等待的线程，然后抛出异常。

&emsp;&emsp;`CyclicBarrier`还可以使用`reset()`来主动介入使栅栏重置：

```java
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
```

&emsp;&emsp;通过`breakBarrier()`破坏当前回合栅栏，并重新开启新的回合。

&emsp;&emsp;`CyclicBarrier`相对来说是比`CountDownLatch`复杂的，巧妙地使用`ReentrantLock`和`Condition`实现了阻塞、计数，并且可以重用，相比`CountDownLatch`更灵活，还可以使到达栅栏后先执行指定的触发任务再执行后续任务。
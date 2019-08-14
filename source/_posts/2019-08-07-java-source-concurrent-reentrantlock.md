---
title: Java源码学习：ReentrantLock学习
description: 学习AQS后趁热打铁学习下可重入锁
categories:
 - 后端
 - Java
tags:
 - Java
 - 并发
---



&emsp;&emsp;前面学习了AQS，趁热打铁继续学习基于AQS实现的可重入锁`ReentrantLock`。

使用`ReentrantLock`的方式也很简单，通常都是以下的方式来使用：

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
// 代码
lock.unlock();
```

### 关于`lock.lock()`

&emsp;&emsp;实现即为调用`Sync`对象的`lock()`方法，而可重入锁的同步控制基础就是内部类`Sync`，而`Sync`就是继承自`AbstractQueuedSynchronizer`实现的，使用AQS的`state`来表示持有锁的线程的数量。`Sync`的子类有`FairSync`和`NonfairSync`，也就是常听说的可重入锁的公平锁版本和非公平锁版本，通过在构造方法里传入true或false来选择使用公平锁或者非公平锁。

在默认情况使用无参构造方法，创建的是非公平锁即`NonfairSync`，

#### 先看下非公平锁的`lock()`：

```java
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

&emsp;&emsp;可以看到，先通过AQS中的CAS操作`compareAndSetState`在期望是0的情况下尝试将state状态设置为1，如果成功了，也就是说明了AQS中的锁是无线程持有的，那么就调用`setExclusiveOwnerThread()`将当前线程设置为独占模式下拥有锁的线程，即顶层父类`AbstractOwnableSynchronizer`的`Thread exclusiveOwnerThread`属性；如果CAS操作失败，说明锁被其他线程持有，那么就调用`acquire(1)`进行尝试获取锁，可能会将当前线程插入同步队列队尾，这里在前面学习AQS的笔记中已经解析过。

#### 而对于公平锁`FairSync`的`lock()`：

```java
        final void lock() {
            acquire(1);
        }
```

&emsp;&emsp;则是只调用`acquire(1)`来参与尝试获取锁。

&emsp;&emsp;前文学习AQS的时候已经知道，`acquire()`会先调用子类实现的`tryAcquire()`方法来尝试获取锁，那么再看下公平与非公平版本是分别如何实现的。

#### 非公平锁`NonfairSync`的`tryAcquire()`：

```java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        
        // Sync.class
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

&emsp;&emsp;`NonfairSync`的`tryAcqiure()`是直接调用父类`Sync`的`nonfairTryAcquire()`方法。再看`nonfairTryAcquire()`方法，先获取当前线程，调用AQS的`getState()`获取同步状态即锁的状态，如果状态c等于0，并且通过CAS设置state成功，则将当前线程设置为独占锁的线程，这里和`NonfairSync#lock()`第一步还是一样的（`所以非公平锁还是挺霸道的，一来就是想直接独占锁，即便第一步失败了，在第二步tryAcquire的时候还是「贼心不死」，还是想尝试再直接独占锁`）；如果同步状态c不为0，再判断当前线程是否是独占锁的线程，如果是，则对状态c进行累加，即标明重入次数，将新的状态值进行设置。在上面两步判断中，如果CAS设置失败则返回false，以及如果当前线程不是锁的独占持有的线程也返回false，都说明尝试获取锁失败，在前文学习AQS已经知道，`tryAcquire()`失败则将当前线程插入同步队列进行等待排队获取锁。


#### 再看公平锁`FairSync`的`tryAcquire()`

```java

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

&emsp;&emsp;同样也是先获取当前线程以及获取锁的状态，如果锁的状态c等于0，即没有线程持有锁，再调用`hasQueuedPredecessors()`判断当前线程之前是否存在排队线程，如果不存在，则通过CAS设置state，如果设置成功，则将当前线程设置为独占锁的线程，返回true；如果锁的状态不为0，并且当前线程就是独占锁的线程，那么和`NonfairSync`的`tryAcqiure()`做法相同，不做赘述。


### 关于`lock.unlock()`

&emsp;&emsp;实现即调用`sync.release(1)`，由于`Sync`并没重写`release()`方法，所以无论公平锁还是非公平锁都是调用的AQS的`release()`：

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

AQS再调用子类`Sync`实现的`tryRelease()`：

```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

&emsp;&emsp;先将锁的状态减去释放的数量，这里是减1，再判断当前线程是否是独占锁的线程，如果不是，就抛出异常；再判断减1之后的锁状态值是否等于0，如果等于0，说明完全释放了锁，则将独占锁的线程设为空；继续再设置锁状态，返回释放成功与否，只有锁状态为0即完全释放了锁时，才会返回true。

&emsp;&emsp;再回到`release()`中，`tryRelease()`释放成功之后，调用`unparkSuccessor()`唤醒后继等待的线程，前文学习AQS已学习了，不再赘述。

&emsp;&emsp;通过对比，可以发现，公平锁获取锁还需要先判断同步队列中是否存在排队的线程，按照等待顺序来获取锁，而非公平锁则没这个限制，是更「霸道」的抢占，而`ReentrantLock`默认也是非公平锁，或许是因为非公平锁这种特征会效率更高。

### 其他方法

#### `tryLock()`

```java
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
```

&emsp;&emsp;对于不带等待时间参数的`tryLock()`，其实公平锁和非公平锁都是调用非公平的`nonfairTryAcquire()`，而带有等待时间参数的`tryLock()`:

```java
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
    // AQS 中的方法：
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```

&emsp;&emsp;可以看到这种带有等待时间的，先调用AQS的`tryAcquireNanos()`，在其中，再根据公平锁还是非公平锁调用相应的`tryAcquire()`实现，如果`tryAcquire()`尝试获取锁失败，则调用`doAcquireNanos()`以独占定时的模式获取锁:

```java
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

&emsp;&emsp;其实对比`acquireQueued()`方法，可以发现，相差不大，只是多了对时间的判断，在`doAcquireNanos()`中，先计算出deadline，也就是设置的定时的结束时间，然后就是和`acquireQueued()`类似的将当前线程加入等待队列，后面在死循环中遍历前驱节点，直到前驱节点是头节点并且尝试获取锁成功，在循环过程中加了对时间的判断，如果超过了设置的定时结束时间，则直接返回false退出。
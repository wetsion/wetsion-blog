---
title: Java源码学习：Semaphore学习
description: 继续学习基于AQS实现的信号量Semaphore
categories:
 - 后端
 - Java
tags:
 - Java
 - 并发
---



前面在学习了AQS之后又学习了下`ReentrantLock`，接下来再来看下`Semaphore`。

`Semaphore`就是信号量，通常用于限制线程数，控制同时访问资源的线程数量。也是基于AQS来实现的，和`ReentrantLock`类似，也分公平锁和非公平锁，默认也是使用非公平锁。

先通过一个买票的例子来练习下，设定有两个卖票窗口，有十个人要买票，可以使用`Semaphore`来实现：

```java
public class SemaphoreDemo {

    static class SemphoreRunner implements Runnable{
        private Semaphore semaphore;
        private int user;
        public SemphoreRunner(Semaphore semaphore, int user){
            this.semaphore = semaphore;
            this.user = user;
        }
        @Override
        public void run() {
            try {
                semaphore.acquire();
                System.out.println("用户：" + user + "进入窗口，准备买票");
                Thread.sleep((long) (Math.random() * 3000));
                System.out.println("用户：" + user + "买好票，准备离开");
                Thread.sleep((long) (Math.random() * 3000));
                System.out.println("用户：" + user + "已经离开");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                semaphore.release();
            }
        }
    }

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(2);
        ExecutorService service = Executors.newCachedThreadPool();
        for (int i = 0; i < 10; i++){
            service.execute(new SemphoreRunner(semaphore, i+1));
        }
        service.shutdown();
    }
}
```

运行结果：

```java
用户：1进入窗口，准备买票
用户：2进入窗口，准备买票
用户：1买好票，准备离开
用户：1已经离开
用户：3进入窗口，准备买票
用户：2买好票，准备离开
用户：3买好票，准备离开
用户：3已经离开
用户：4进入窗口，准备买票
用户：2已经离开
用户：5进入窗口，准备买票
用户：4买好票，准备离开
用户：4已经离开
用户：6进入窗口，准备买票
用户：5买好票，准备离开
用户：5已经离开
用户：7进入窗口，准备买票
用户：6买好票，准备离开
用户：6已经离开
用户：8进入窗口，准备买票
用户：7买好票，准备离开
用户：8买好票，准备离开
用户：8已经离开
用户：9进入窗口，准备买票
用户：7已经离开
用户：10进入窗口，准备买票
用户：9买好票，准备离开
用户：9已经离开
用户：10买好票，准备离开
用户：10已经离开
```

使用`Semaphore`，必须先调用`acquire()`获取锁（或者说是许可，这里我还是称呼其为锁，只不过`Semaphore`维护了多把锁），先看下`acquire()`源码：

### `acquire()`

```java
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    // AQS :
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

这里sync是内部类`Sync`对象，是AQS在`Semaphore`的抽象实现，这里`Sync`和`ReentrantLock`中的`Sync`类似，也有两个子类实现，即公平与非公平两个版本，这里调用`sync.acquireSharedInterruptibly(1)`实际上是调用AQS的`acquireSharedInterruptibly()`方法，从前面学习已经知道，`acquireSharedInterruptibly()`是在共享模式下的获取锁（会被线程中断打断），先调用AQS子类实现的`tryAcquireShared()`来尝试获取锁，这里就是调用`Semaphore`中`Sync`的`tryAcquireShared()`，分为了公平和非公平两个实现版本，先看下默认的非公平实现：

#### 非公平的`tryAcquireShared()`

```java
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
    // Sync#nonfairTryAcquireShared()
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
```

在非公平的实现中，调用`Sync`默认提供了的`nonfairTryAcquireShared()`方法，在`nonfairTryAcquireShared()`中先获取AQS中的同步状态state（这里state的是在`Semaphore`对象new的时候通过`Sync`调用AQS的`setState`设置的），也就是可用的锁的个数，拿state减去要获取的锁个数（默认为1），如果小于0，说明目前没有剩余可用锁，则直接返回负值；如果大于0，则再通过CAS修改state的值，修改为减之后的值，如果成功，则返回剩余锁个数（即获取锁成功了），如果失败则继续循环。

再看下公平的实现：

#### 公平的`tryAcquireShared()`

```java
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

这里也是自旋，先通过`hasQueuedPredecessors()`判断当前线程之前是否存在排队线程，如果有排队等待的线程，则不参与获取锁的竞争，返回-1；如果没有，则同样获取当前同步状态state，即可用锁个数，减去要获取的锁个数，如果小于0，即目前无剩余可用锁，直接返回负值；如果大于0，和非公平实现一样，通过CAS修改state，如果成功，返回剩余锁个数（为正数），失败则继续自旋循环。


接上文，在AQS的`acquireSharedInterruptibly()`中，如果`tryAcquireShared()`大于0，即获取锁成功，如果为负值，即获取锁失败，调用`doAcquireSharedInterruptibly()`方法，这里在前文AQS的学习中已知晓，将当前线程插入到等待队列队尾排队等待唤醒，不再赘述。

对于`acquire()`获取锁的操作，Semaphore的公平和非公平实现又体现在哪，有什么区别呢，通过通读源码，不难发现，公平的实现比非公平的实现多了两行代码，即`if (hasQueuedPredecessors()) return -1;`，公平的实现会先判断在等待队列里是否有线程在排队等待，而非公平的实现则直接参与竞争锁。



### `release()`

`Semaphore`的`release()`方法直接调用AQS的`releaseShared()`：

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

再调用`Semaphore`中`Sync`的`tryReleaseShared()`实现，这里非公平和公平都共用该方法释放锁：

```java
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }
```

这里实现也比较简单，即计算释放锁之后可用锁的数量，然后CAS操作将state更新成新的值，成功后，在调用AQS的`doReleaseShared()`唤醒后续等待的线程，不再赘述。

`Semaphore`作为信号量，可以使用在竞争资源有限的场景，但讲道理，实际开发中，却是没用到过 -.-||
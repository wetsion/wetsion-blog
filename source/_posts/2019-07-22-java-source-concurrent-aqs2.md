---
title: Java源码学习：AbstractQueuedSynchronizer(AQS)学习(二)
description: AbstractQueuedSynchronizer关于条件的学习
categories:
 - 后端
 - Java
tags:
 - Java
 - 并发
---



&emsp;&emsp;上一篇笔记学习了AQS的独占锁与共享锁，这篇笔记继续学习AQS。

&emsp;&emsp;AQS中还有个`ConditionObject`类，`ConditionObject`实现了`Condition`接口，`Condition`提供了一系列等待和通知的方法，例如`await()`、`awaitUninterruptibly()`、`signal()`、`signalAll()`等，`Condition`是用来实现线程之间的等待，***而且`Condition`对象只能在独占锁中使用***。

&emsp;&emsp;同样，先看下`ConditionObject`中有哪些属性：

- `private transient Node firstWaiter` 条件队列的头节点
- `private transient Node lastWaiter` 条件队列的尾节点

&emsp;&emsp;`ConditionObject`类中包含了这样两个属性，中维护一个`Node`节点构成的条件队列，而`Node`中又有一个`nextWaiter`，说明了条件队列是单向队列。


&emsp;&emsp;看下`await()`是如何实现让当前持有锁的线程阻塞等待释放锁的：

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

&emsp;&emsp;可以看到`await()`不会忽略中断，如果当前线程被中断，则抛出异常；调用`addConditionWaiter()`往条件队列添加一个等待节点：

```java
    private Node addConditionWaiter() {
        Node t = lastWaiter;
        // If lastWaiter is cancelled, clean out.
        if (t != null && t.waitStatus != Node.CONDITION) {
            unlinkCancelledWaiters();
            t = lastWaiter;
        }
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        if (t == null)
            firstWaiter = node;
        else
            t.nextWaiter = node;
        lastWaiter = node;
        return node;
    }
    private void unlinkCancelledWaiters() {
        Node t = firstWaiter;
        Node trail = null;
        while (t != null) {
            Node next = t.nextWaiter;
            if (t.waitStatus != Node.CONDITION) {
                t.nextWaiter = null;
                if (trail == null)
                    firstWaiter = next;
                else
                    trail.nextWaiter = next;
                if (next == null)
                    lastWaiter = trail;
            }
            else
                trail = t;
            t = next;
        }
    }
```

&emsp;&emsp;这里先获取条件队列的尾节点，如果尾节点的状态不为`CONDITION`，则调用`unlinkCancelledWaiters()`（从条件队列头节点开始遍历，如果节点的状态不为`CONDITION`，将节点从条件队列中移除）；根据当前线程创建状态为`CONDITION`的新节点；如果尾节点不存在，将新节点设置为头节点，否则将新节点设置为尾节点的后继节点，同时将新节点设置为新的尾节点（ ***总结一句话就是新建条件节点插入到条件队列队尾*** ）。

&emsp;&emsp;再回到`await()`接着看，新建完节点之后，调用`fullyRelease()`：

```java
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

&emsp;&emsp;获取当前state状态值，使用这个状态调用`release()`释放独占锁并唤醒同步队列中下一个等待节点，释放成功，将当前节点的状态设置为`CANCELLED`，失败就抛出异常。

&emsp;&emsp;通过`fullRelease()`释放锁之后，使用`isOnSyncQueue()`判断当前节点是否在同步队列中，如果不在同步队列，则阻塞当前线程，如果已经在同步队列，调用`acquireQueued()`获取锁，然后当前节点的后继节点如果不为空，说明条件队列该节点后面有在等待的线程，则调用`unlinkCancelledWaiters()`将条件队列中不为`CONDITION`中的节点移除，最后，如果interruptMode为THROW_IE则抛出异常，如果是REINTERRUPT，则中断当前线程。


再看`signal()`：

```java
    public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }
    private void doSignal(Node first) {
        do {
            if ( (firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
                 (first = firstWaiter) != null);
    }
    final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

&emsp;&emsp;这里先通过子类实现重写的`isHeldExclusively()`来判断当前线程是否独占锁，如果未持有锁，则抛出异常，然后从条件队列的头节点firstWaiter开始遍历，将头节点的后继节点设置为新的头节点，如果新的头节点为空，那说明条件队列是个空队列了，那么再将尾节点也设置为null，为了方便GC，再将`first.nextWaiter`引用设置为null，在do-while循环条件中，调用`transferForSignal()`，将头节点的状态通过CAS更新为默认的0（如果更新失败，说明该节点状态不为`CONDITION`了，即已经在同步队列里了，返回false退出，继续`doSignal()`的循环），然后调用`enq(node)`将节点插入到同步队列中，在上一篇文章说独占模式的时候，已经知道`enq()`返回的是插入之前时，同步队列的尾节点，而这个尾节点也是当前插入节点的前驱节点，而插入的节点自然成为新的尾节点，然后获取之前尾节点也就是当前节点前驱节点的`waitStatus`，如果waitStatus大于0也就是`CANCELLED`状态，或者无法通过CAS设置成`SIGNAL`状态，则调用`Lock.unpark()`唤醒当前节点中的线程，然后退出`doSignal()`中的循环。

&emsp;&emsp;从上面的源码可以发现，`signal()`唤醒并不是立即唤醒，而是将条件队列的节点插入到同步队列尾部等待，还是需要等待前面的节点获取完锁之后，才会轮到它。

***总结***

&emsp;&emsp;其实对于`await()`和`signal()`，我觉得可以用几句话就能简单总结，在独占模式下，多个线程争抢锁，某个线程A获取到了锁（其他线程由于阻塞进入到同步队列中），而这个线程可以使用`await()`释放锁，阻塞自己，将当前线程添加到条件队列中（注意这里是条件队列而不是同步队列，就意味着这个线程将不能再参与争抢锁），而这时，其它在同步队列中的线程将获取锁，在某一个线程B获取到锁之后，它也可以继续`await()`释放锁，把自己放到条件队列中，也可以通过`signal()`将在条件队列中第一个线程唤醒，将其加入到同步队列参与竞争锁（或者通过`signalAll()`将条件队列中的所有线程唤醒，都加入到同步队列中参与竞争锁）。

&emsp;&emsp;用最近在玩的「云顶之弈」游戏来比喻的话（可能不太贴切），同步队列好比放在棋盘上等待参与战斗的英雄们，而条件队列相当于在棋盘下方放置的备用英雄们，棋盘上的英雄竞争攻击的机会，`await()`就相当于玩家通过把英雄放回到下方备用英雄区，而`signal()`就相当于把备用区的英雄放到棋盘上参与战斗。

&emsp;&emsp;再写个栗子加深下理解：

```java
public class ConditionDemo {

    static class Thread1 implements Runnable{
        private Lock lock;
        private Condition condition;
        public Thread1(Lock lock, Condition condition) {
            this.lock = lock;
            this.condition = condition;
        }
        @Override
        public void run() {
            try {
                System.out.println("线程1争抢锁");
                lock.lock();
                System.out.println("线程1获得锁，调用await释放锁");
                condition.await();
                System.out.println("线程1被唤醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }
    static class Thread2 implements Runnable{
        private Lock lock;
        private Condition condition;
        public Thread2(Lock lock, Condition condition) {
            this.lock = lock;
            this.condition = condition;
        }
        @Override
        public void run() {
            try {
                System.out.println("线程2争抢锁");
                lock.lock();
                System.out.println("线程2获得锁, 调用await释放锁");
                condition.await();
                System.out.println("线程2被唤醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }
    static class Thread3 implements Runnable{
        private Lock lock;
        private Condition condition;
        public Thread3(Lock lock, Condition condition) {
            this.lock = lock;
            this.condition = condition;
        }
        @Override
        public void run() {
            try {
                System.out.println("线程3争抢锁");
                lock.lock();
                System.out.println("线程3获得锁");
                condition.signalAll();
                System.out.println("线程3唤醒条件队列等待线程");
            } finally {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        Thread thread1 = new Thread(new Thread1(lock, condition));
        Thread thread2 = new Thread(new Thread2(lock, condition));
        Thread thread3 = new Thread(new Thread3(lock, condition));

        thread1.start();
        thread2.start();
        Thread.sleep(5000L);
        thread3.start();
    }
}
```

&emsp;&emsp;这里定义了三个线程，线程一和二先来争抢锁，然后都使用`await()`，然后主线程等待5秒后再启用线程三，线程三中使用`signalAll()`将条件队列中所有线程唤醒，运行结果如下：

```java
线程1争抢锁
线程1获得锁，调用await释放锁
线程2争抢锁
线程2获得锁, 调用await释放锁
线程3争抢锁
线程3获得锁
线程3唤醒条件队列等待线程
线程1被唤醒
线程2被唤醒
```

&emsp;&emsp;可以看到，线程1先获取到锁，然后`await()`释放锁之后先进入条件队列，所以再最后唤醒时也是先唤醒加入到同步队列，先获取锁。

&emsp;&emsp;这篇笔记又学习了AQS的条件队列Condtion这一块，后面笔记将再结合具体的AQS实现类来学习AQS以及并发各种同步的实现。

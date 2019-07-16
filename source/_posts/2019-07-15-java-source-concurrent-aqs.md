---
title: Java源码学习：AbstractQueuedSynchronizer(AQS)学习
description: AbstractQueuedSynchronizer(AQS)学习
categories:
 - 后端
 - Java
tags:
 - Java
 - 并发
---


&emsp;&emsp;最近在学习JUC包下相关类时，例如如`ReentrantLock`、`Semaphore`等，发现内部都是基于Java提供的AQS框架来实现的，也就是`AbstractQueuedSynchronizer`类，所以先学习搞懂此类，再去看那些类，也会有助于理解。

&emsp;&emsp;AQS即`AbstractQueuedSynchronizer`，是JUC提供的一个框架，`用于实现依赖于先进先出（FIFO）等待队列的阻塞锁和相关同步器（信号量，事件等），此类旨在成为依赖单个原子int值来表示状态的大多数同步器的有用基础`，正如这个类的注释说的，`此类支持默认独占模式和共享模式之一或两者`，即独占锁和共享锁。

&emsp;&emsp;AQS常见的使用方式就是定义内部类继承它，将该内部类作为一个辅助类来实现同步。

&emsp;&emsp;在类注释一开始就提到了依赖于一个FIFO队列，这个队列是一个同步队列，在类的内部有一个`Node`静态内部类，该类就是CLH队列中节点的实现，看一下源码：

```java
static final class Node {

        static final Node SHARED = new Node();
   
        static final Node EXCLUSIVE = null;

        static final int CANCELLED =  1;
       
        static final int SIGNAL    = -1;
        
        static final int CONDITION = -2;
        
        static final int PROPAGATE = -3;

        volatile int waitStatus;

        volatile Node prev;

        volatile Node next;

        volatile Thread thread;

        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

可以看到具有以下属性，

&emsp;&emsp;节点中包含前驱节点`prev`和后继节点`next`，说明这个同步队列是一个双向队列；

&emsp;&emsp;`waitStatus`表示当前节点的状态，默认状态值为0（condition时默认初始化为`CONDITION`），一共有五个状态，除了默认0之外，还有`SIGNAL`(-1)、`CANCELLED`(1)、`CONDITION`(-2)、`PROPAGATE`(-3)：

- `CANCELLED` 代表线程已取消，由于超时或者被打断，具有该状态的节点不会再被阻塞
- `SIGNAL` 代表此节点的后续的节点是被阻塞（或将被阻塞）的，那么当当前节点释放或者取消的时候，就需要unpark唤醒它的后继节点。
- `PROPAGATE` 代表releaseShared应该传播到其他节点，仅在`doReleaseShared()`中设置，也仅限头节点，以确保继续传播。
- `CONDITION` 代表当前节点线程处在条件队列中（这个是一个特殊状态，只在condition队列即等待队列中节点中存在，同步队列中不存在这种状态的节点）

&emsp;&emsp;`thread`属性代表当前节点持有的线程，或者说拥有当前节点的线程

&emsp;&emsp;`nextWaiter`属性是链接在条件队列等待的下一个节点，或者是特殊值SHARED。如果是特殊在SHARED，所在当前节点是共享模式；如果是null，代表所在的当前节点是独占模式，如果是其他值，所在的当前节点也是独占模式，但`nextWaiter`将是condition条件队列的下一个节点。


&emsp;&emsp;通过看`AbstractQueuedSynchzonizer`类的注释和通览整个类，虽然是个抽象类，但并没有一个抽象方法，而需要子类重写实现的方法都是通过在方法体抛出`UnsupportedOperationException`异常来让子类知道，提供了以下五个方法给子类实现

- tryAcquire

> 尝试在独占模式下acquire，子类实现方法应查询对象的状态state字段是否允许以独占模式acquire，如果允许，那么可以acquire。该方法通常在线程执行acquire时（即执行`acquire()`方法）调用，如果失败，则`acquire()`方法会将线程加入等待队列（如果还没加入等待队列），直到它被其他线程发出的信号释放

- tryRelease

> 尝试在独占模式下设置状态来体现对节点的释放，通常在线程执行`release()`方法释放节点时调用

- tryAcquireShared

> 尝试在共享模式下acquire，子类实现方法应查询对象的state状态字段是否允许acquire，如果允许，可以acquire。方法通常在线程执行acquire时（即执行`acquireShared()`）调用，如果失败，则`acquireShared`会将线程加入等待队列（如果线程还没加入等待队列），直到被其他线程发出的信号释放

- tryReleaseShared

> 尝试在共享模式下设置状态来体现对节点的释放，通常在线程执行`releaseShared()`方法释放节点时调用

- isHeldExclusively

> 判断当前同步器是否仅被当前线程独占。需要注意的是，该方法仅被`AbstractQueuedSynchronizer.ConditionObject`中的方法调用，因此不使用condition条件，则不需要实现。


&emsp;&emsp;除了内部类`Node`之外，还有一个`ConditionObject`，这里先暂不去看它。再看`AbstractQueuedSynchronizer`还有哪些属性：

- `private transient volatile Node head;`

> 同步队列的头节点，如果head存在，head节点的`waitStatus`属性确保不会成为`CANCELLED`

- `private transient volatile Node tail;`

> 同步队列的尾节点 ，仅会在`enq()`方法新增新的等待节点时被修改，即每当新节点进来都会被插到最后

- `private volatile int state;`

> 同步状态，也可以理解为锁的状态，0代表没被占用，大于0代表有线程持有当前锁


&emsp;&emsp;同时对于这些属性以及`Node`类中的属性，除了使用volatile修饰之外，AQS还提供了对应的CAS操作，来保证以原子方式来将这些属性设置为给定的更新值，例如`compareAndSetHead()`、`compareAndSetWaitStatus()`。


&emsp;&emsp;前面通过注释了解到AQS类支持独占模式（独占锁）和共享模式（共享锁），在独占锁下，其他线程试图获取锁就无法成功，在共享锁下，多个线程获取锁可能会成功。

先看一下如何实现独占锁。

### 独占模式（独占锁）

&emsp;&emsp;独占锁，顾名思义，就是在多个线程尝试获取锁的时候，只能有一个线程能获取到锁，而其他线程将会被阻塞等待（按照FIFO排队等待），而持有锁的线程释放锁之后，将会唤醒正在等待锁的下一个线程。

而***获取独占锁***，主要依靠`acquire()`方法和`acquireInterruptibly()`：


```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```

&emsp;&emsp;`acquireInterruptibly()`是以独占可中断模式获取锁，而`acquire()`也是独占模式获取锁，但会忽略中断。`acquireInterruptibly()`会先判断线程是否中断，若中断就抛出异常。

&emsp;&emsp;两个方法都会首先通过`tryAcquire()`尝试获取锁，`tryAcquire()`方法前面说过，是留给子类实现的，用于查询state的值来判断允许以独占模式获取锁，如果可以则返回true，即代表当前线程获得了锁，`acquire()`方法直接返回，可以执行当前线程要做的操作。如果返回false，说明当前线程没有获得锁，

&emsp;&emsp;`acquireInterruptibly()`将会接着调用`doAcquireInterruptibly()`（其实查看`doAcquireInterruptibly()`会发现其实就是等同于`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`，不同点是`acquireQueued()`中`if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())` 后只是 `interrupted = true`，而`doAcquireInterruptibly()` 却将异常抛了出去`throw new InterruptedException()`，具体看下面的源码学习）

&emsp;&emsp;而`acquire()`则是接着继续执行`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`，这里先执行`addWaiter(Node.EXCLUSIVE)`：

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

&emsp;&emsp;这里传入的参数为`Node.EXCLUSIVE`，通过注释可知是代表是独占模式，在`addWaiter()`中将当前线程包装成node节点，然后获取`tail`尾节点。1.如果有尾节点，将当前node节点的前驱节点指向尾节点，再通过CAS操作，将当前node节点设置为新的尾节点，成功后再将老的尾节点的后继节点指向到当前node节点（也就是新的尾节点），通过这番操作，***新增的节点都会作为尾节点插入到队列中***，并构成了一个双向队列。2.如果没尾节点（说明此时这是一个空队列），则调用`enq()`方法：

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

&emsp;&emsp;可以看到这里是个for(;;)死循环，还是先获取尾节点，如果尾节点为空，则创建一个无任何的Node节点，并通过CAS操作设置成`head`头节点，再将头节点设置到尾节点，进入下一次循环，这样就能获取到尾节点了，将当前节点的前驱节点指向尾节点（由于前面将头节点指向了尾节点，所以此时当前节点的前驱节点也就是头节点），再通过CAS操作将当前节点设置为新的尾节点，再将旧的尾节点的后继节点指向当前节点（也就是头节点的后继节点指向了当前节点），这样也将当前节点插到了队尾，并也构成了一个双向队列。

&emsp;&emsp;再回到前面，`addWaiter()`当前节点插入队尾成功后，再执行`acquireQueued()`：

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed) cancelAcquire(node);
        }
    }
    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```

&emsp;&emsp;这个方法用于让在队列中的线程以独占模式获取锁，通常被condition wait方法（`ConditionObject`中的方法）以及`acquire()`方法调用。可以看到依然是for(;;)死循环，首先获取当前节点的前驱节点，1.如果前驱节点是头节点，并且通过`tryAcquire()`获取锁成功，则将当前节点设置为头节点，并将当前节点中的线程清空，当前节点的前驱节点也设为空（这里`p.next=null`是为了将老的头节点的后继清空，这样老的节点对象就没有任何依赖关系了，便于GC回收对象），而`interrupted`还是false，说明不被中断，在`acquire()`将不会执行到`selfInterrupt`，所以当前线程将继续执行。2.如果前驱节点不是头节点获取`tryAcquire()`获取锁失败，2.先通过`shouldParkAfterFailedAcquire()`检查是否需要阻塞获取锁失败的节点：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

&emsp;&emsp;在`shouldParkAfterFailedAcquire()`中，检查前驱节点的状态（waitStatus），如果前驱节点为`SIGNAL`，返回true，即当前节点需要被阻塞（`SIGNAL`前文已说代表后继节点是被阻塞或将被阻塞的）；如果状态大于0，也就是状态是`CANCELLED`（只有`CANCELLED`是大于0的），则循环不断向前寻找节点，直到节点的waitStatus不大于0也就是不等于`CANCELLED`，找到节点后，设置为前驱节点，并与之构成双向队列，返回false，即当前节点不需要被阻塞；如果状态既不是`SIGNAL`，也不大于0，则通过CAS将前驱节点的waitStatus设置为`SIGNAL`，然后返回false，即当前节点不需要被阻塞。

&emsp;&emsp;再回到上面`acquireQueued()`中，如果`shouldParkAfterFailedAcquire()`返回true，则调用`parkAndCheckInterrupt()`让当前线程阻塞，再将`interrupted`设为true，但此时并没跳出for循环，将会继续获取前驱节点，然后判断是否是头节点以及尝试获取锁（需要注意的是，这时的前驱节点和上一次循环时的前驱节点可能会不一样，因为在`shouldParkAfterFailedAcquire()`中有向前寻找节点并重新构成双向队列的操作）。当发生异常，非正常退出时，调用`cancelAcquire()`将当前节点状态设置为CANCELLED，即所在线程已取消，不需要唤醒了。


以上是独占模式下获取锁，那么释放锁是如何做的呢。通过调用`release()`方法开始***释放锁***：

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

&emsp;&emsp;这里先调用`tryRelease()`方法尝试释放锁，该方法由子类去实现，表示当前持有锁的线程动作执行完了，需要释放锁，将锁让给其他线程。`tryRelease()`成功之后，获取头节点，如果头节点不为空，并且头节点的状态不为0，说明队列中可能存在需要唤醒的等待节点

> 注一：为什么不为0就说明队列存在需要唤醒的等待节点呢，我的理解是，一方面，首先头节点如果存在就不会是`CANCELLED`，而`CONDITION`仅在条件队列中，这里是同步队列，`PROPAGATE`又仅在`doReleaseShared()`中设置，如果head头节点的状态不为初始化的0，那么就只能为`SIGNAL`。另一方面，在往队列插入节点到队尾成功后，前文在`acquireQueued()`中，可以看到，如果`tryAcquire()`获取锁失败，在`shouldParkAfterFailedAcquire()`会将前驱节点的状态通过CAS更新为`SIGNAL`，然后再调用`parkAndCheckInterrupt()`让当前线程节点阻塞。而`SIGNAL`就指明当前节点的后继节点是被阻塞的，所以即存在需要唤醒的节点

> 注二：回头看整个获取锁的过程，其实也就是节点入队列的过程，头节点在刚初始化是无任何状态的Node对象，但第一个线程节点入队列获取到锁之后，头节点就变成第一个线程节点，此后，头节点就是当前获取锁正在执行的节点。

（接注一注二之前内容），则调用`unparkSuccessor()`唤醒下一个被阻塞的线程节点：

```java
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

&emsp;&emsp;可以看到，获取当前节点的状态，如果小于0（其实就是`SIGNAL`），通过CAS更新为0，然后获取后继节点s，如果后继节点为空的话，就从tail尾节点往前遍历，如果节点的状态小于等于0，就将这个节点指给s，如果s不为空，就通过`LockSupport.unpark()`唤醒后继节点s的线程。


上面是关于独占锁的获取与释放，那么对于共享锁呢，共享模式下，多个线程获取锁都可能会成功。

### 共享模式（共享锁）

与独享锁类似，共享模式下，通过`acquireShared()`和`acquireSharedInterruptibly()`方法获取锁。这两个方法都是以共享模式获取锁：

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

通过对比可以发现，前者会忽略中断，后者首先检查中断状态，被中断则抛出异常中止。

&emsp;&emsp;先看下`acquireShared()`，先调用子类重写实现的`tryAcquireShared()`方法尝试获取共享锁，在独享模式时，`tryAcquire()`返回的就是true或false，而这里返回的是int数，如果获取失败，则返回负数，如果在共享模式下获取锁成功，但没有后续的共享模式获取可以成功（可以理解为锁的获取名额用完了），则返回0，如果共享模式下获取锁成功，并且后续的共享模式下获取锁也可能可以成功（即获取锁的名额还有），则返回正数。所以如果`tryAcquireShared()`返回小于0，即获取锁失败，那么调用`doAcquireShared()`：

```java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

&emsp;&emsp;这里和独占模式有些类似，独占模式是`addWaiter(Node.EXCLUSIVE)`，共享模式下是`addWaiter(Node.SHARED)`，从前文已知，`addWaiter()`是将当前线程构建成节点，并插入到队列中，这里同理。然后进入到for循环里。先获取当前线程节点的前驱节点，如果前驱节点就是头节点的话，调用`tryAcquireShared()`尝试获取锁（这里和独占模式是类似的，只有前驱节点是head头节点才能尝试获取锁），如果返回大于等于0，说明获取共享锁成功，接着调用`setHeadAndPropagate()`：

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

&emsp;&emsp;与独占锁有些类似，获取锁成功后，调用`setHeadAndPropagate()`也将当前节点设置为头节点，但不同于独占锁的是，不仅仅将当前节点设置为头节点，还根据`tryAcquireShared()`返回的值判断，如果大于0，获取当前节点的后继节点，如果后继节点也是shared共享模式节点，则通过调用`doReleaseShared()`方法向后传播，唤醒后面等待的节点（在后面释放锁的时候具体来看该方法）。

&emsp;&emsp;再回到`doAcquireShared()`，剩下的和独占模式几乎一样，如果当前节点前驱节点不是头节点，也是调用`shouldParkAfterFailedAcquire()`，`parkAndCheckInterrupt()`。

&emsp;&emsp;那么`doAcquireSharedInterruptibly()`呢，该方法与`doAcquireShared()`几乎一样，只是在`if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())`里抛出了异常:`throw new InterruptedException();`。

再看下***共享模式下锁的释放***：

&emsp;&emsp;AQS提供了`releaseShared()`方法来在共享模式下释放锁：

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

&emsp;&emsp;同独占模式类似，先调用由子类重写实现的`tryReleaseShared()`尝试释放，如果成功，则调用`doReleaseShared()`：

```java
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

&emsp;&emsp;在前文中，获取到锁之后`setHeadAndPropagate()`方法中也调用来该方法，这里详细看下`doReleaseShared()`的实现。也是一个for循环，获取到头节点赋值给节点h，如果头节点的状态为`SIGNAL`，且能通过CAS将头节点的状态设置为0，则调用`unparkSuccessor()`唤醒头节点的后继节点，如果不能通过CAS将头节点状态设置0，则进行下一轮循环重试；如果头节点状态为0，就调用CAS将状态设置为`PROPAGATE`（开头时候已知道，设置为`PROPAGATE`即以确保可以向后传播），如果设置失败，就进行下一轮循环重试；如果h等于头节点，即头节点未发生变化，则跳出循环。

&emsp;&emsp;前文说到，在AQS中除了`Node`类，还有个`ConditionObject`类，本文先学习到这里，下一篇文章继续学习`ConditionObject`，学习AQS如何利用`ConditionObject`来实现等待通知，以及AQS中的其他方法与细节的补充。



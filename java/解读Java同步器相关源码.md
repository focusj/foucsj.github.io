# 解读Java同步器相关源码

关于解读Java同步器相关源码的文章已经数不胜数, 但是经典的东西总能经的起反复的解读和学习. 最好每个人都能亲自翻着源码看一看, 肯定会有收获. 如果因为这篇文章促成你看源码, 那么这篇文章远超过了它内容的价值.

文章包含五部分内容: AQS源码解读；ReentrantLock源码解读；ReentrantReadWriteLock源码解读；CountDownLatch源码解读；拜神仪式. 源码部分并不是面面俱到, 只是大致分析了AQS如何在每个类中发挥作用. 

## AQS部分

AbstractQueuedSynchronizer是基于自旋和先进先出队列的同步器. Java并发包中的可重入锁(ReentrantLock), 读写锁(ReentrantReadWriteLock), 信号量(Semaphore), 闭锁(CountDownLatch)都基于AQS构建. 

1. AQS内部维护了一个先进先出的队列, 组成队列的节点由Node表示, Node大致结构如下: 
```java
static final class Node {
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;


    volatile int waitStatus;

    // 指向前后节点 
    volatile Node prev;
    volatile Node next;

    // 指向当前线程(或者正在获取锁, 或者正在申请信号量...)
    volatile Thread thread;
}
```

其中waitStatus表示当前Node的状态. 举个例子, 假如一个竞争非常激烈的锁, 某个线程一段时间内未竞争到锁, 而取消了. 这是Node会被置为取消的状态. Node初始化时的值为0.

CANCELLED: Thread已取消
SIGNAL: Thread正等待被unpark(获取锁未成功进入队列等待的线程, 会被标记成SIGNAL, 线程也会通过LockSupport.park被挂起)
CONDITION: Thread正在等待Condition, 在condition queue中
PROPAGATE: 只可能头节点被设置该状态, 在releaseShared时需要被传播给后续节点.

2. 在AQS中还有一个非常重要的状态属性: `private volatile int state`. 这个属性可以被子类扩展成不同的用途. 在ReentrantLock中state表示获取锁的数量(最多2147483647个), state=0表示锁未被获取过. ReentrantReadWriteLock, 更是巧妙的把state分成两段来用, 高16位表示读锁数量, 低16位表示写锁数量.因此ReentrantReadWriteLock限制最大的锁数量是65535. Semphore表示剩余的许可数量. CountDownLatch表示闭锁数量.

3. AQS作为一个基类出现, 如果基于它来构建同步工具需要重新定义以下方法:
```
 tryAcquire // 独占模式下获取锁
 tryRelease // 独占模式下释放所
 tryAcquireShared // 共享模式下获取锁
 tryReleaseShared // 共享模式下释放锁
 isHeldExclusively // 判断锁是不是被当前线程独占
```

独占锁只能由一条线程正在使用锁, 共享锁多个线程可以同时使用.

## ReentrantLock 部分

这一部分来分析ReentrantLock的非公平模式下获取和释放锁的过程. 先来说获取锁的过程, 获取锁可以大致的分为两个阶段: 抢占式获取, 如果获取成功则立即返回；排队等待. 下边来看源码: 

```java
// 使用lock获取锁的时候, 如果锁处于空闲状态, 则获取成功立即返回.
// 如果所被当前线程持有, 则嵌套的获取可冲入锁.
// 如果锁被其他线程持有, 则当前线程被切换出去, 直到获取了锁.
final void lock() {
    // 如果当前锁处于空闲状态, 则直接获取锁. 是不用排队的.
    if (compareAndSetState(0, 1)) 
        // 设置当前线程为独占线程
        setExclusiveOwnerThread(Thread.currentThread()); 
    // 如果当前不能获取, 则和公平锁一样进入队列
    else 
        acquire(1);
}

// AQS
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

```

先尝试立即获取锁, 如果刚好抢上了, 则立即返回. 这时可能有其它线程等待, 也可能没有. 获取锁通过CAS修改AQS中state的状态. 如果修改成功, 表示获取锁成功, 设置当前线程为owner.

非公平锁有着更好的性能, 想象一种情况, A线程释放锁, B线程正在等待被唤醒, 而这时C快速的获取锁并释放, 这一切都赶在B被唤醒之前. 而公平锁则无法达到这种共赢的局面. 

接下看插队失败后如何处理: 
```java

public final void acquire(int arg) { // arg = 1
    // 如果获取锁失败 && 并且当前线程被中断当前线程
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 调用线程的interupt方法.
        selfInterrupt();
}

```

上边我们说过tryAcquire需要实现者重写, 我们拿到ReentrantLock.NonfaitLock的实现: 

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

这个方法调用的是父类Sync的nonfairTryAcquire方法. Sync继承了AQS, 而NonfaitLock继承自Sync. 

```java

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取状态
    int c = getState();
    // 空闲状态可以直接获取
    if (c == 0) {
        // 获取成功, set 线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前线程已经获取了锁, 则可重入获取
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 更新当前锁被获取的数量
        setState(nextc);
        return true;
    }
    return false; // 获取失败
}
```
首先, 获取当前AQS的state, 上边说过Reentrant用它来记录锁的获取情况. 如果state == 0, 尝试CAS修改状态, 成功则获取锁成功. 如果state不是0, 则表示当前锁已经被占有, 因为是可重入锁, 所以, 如果当前线程和持有锁的线程是同一个线程, 更新AQS状态, 获取锁成功. 这是第二次插队的过程. 

如果tryAcquire失败了呢? 这次真的要排队了.

```java
// 把当前获取锁的节点放入等待队列
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    // 先尝试直接入队列
    if (pred != null) { 
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 如果失败, 则执行自旋set
    enq(node); 
    return node;
}
```

先尝试看能不能直接一次进队, 如果不行调用enq方法, 执行自旋逻辑去.

最后一次不排队获取锁的机会: 

```java
final boolean acquireQueued(final Node node, int arg) { 
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取该tail的前一个节点
            final Node p = node.predecessor(); 
            // 如果当前节点的上一个节点==头节点, 并且获取锁成功. 
            // 意味着当前的head节点释放了锁. 所以当前线程获取锁成功, 并且把当前节点置为head节点
            if (p == head && tryAcquire(arg)) { 
                setHead(node); 则把node设置为头节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 检查当前线程是否应该被挂起 && 调用LockSupport.park成功则当前线程可被中断.
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
看for循环中的第一个if, 如果node的前一个节点是头节点(即正在占有锁), 并且tryAcquire获取锁成功, 说明当前线程成功的拿到了锁, 不需要被interrupt. 

接着看shouldParkAfterFailedAcquire和parkAndCheckInterrupt两个方法: 
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) // 如果前一个节点的state是SIGNAL, 则当前线程应该interrupt
        /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
        return true;
    if (ws > 0) { // 删除cancelled的node
        /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
如果前一个节点的waitStatus是SIGNAL, 表示这个节点也在等待被唤醒, 当前线程需要被interrupt, 老实排队. 如果waitStatus > 0(这个节点被取消了), 移出队列, 并检查是否有其他节点需要被移除. 否则的话设置前一个节点的waitStatus为SIGNAL. 因为这个方法调用是在acquireQueued方法中的for循环中, 会不停的被调用. 最终期望的结果是, 如果当前线程没有机会获取锁, 则挂起该线程, 并到队列排队. 

parkAndCheckInterrupt方法会调用`LockSupport.park(this)`挂起当前线程. 因为acquireQueued在一个for循环里边, 所以当线程被unpark的时候仍会接着执行重新获取锁的逻辑.

所以回头再看acquire方法就可以用一句话概括: 如果获取成功则立即返回, 如果获取锁失败, 加入等待队列, 然后后调用selfInterrupt方法. 到此, 加锁的过程分析完成. 下边来看unlock的过程. 

```java
public void unlock() {
    sync.release(1);
}

// AQS
public final boolean release(int arg) {
    if (tryRelease(arg)) { // 如果当前线程完全释放锁
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

```
ReentrantLock的unlock方法调用了AQS的release方法. tryRelease我们可以在ReentrantLock.Sync中找到其实现: 

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // 因为是可重入锁, 所以c可能大于1, 所以只有c == 0时, 锁才释放完成.
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c); // 设置state
    return free;
}
```
首先, 判断当前线程获取锁的剩余数量, 如果数量为0, 则释放成功, 把AQS中的Thread置空. 如果数量 != 0, 则把当前的数量set到AQS的state中. 可重入获取锁, 所以可能不会一次性释放完成. 当只有当c为0, 才真正释放, 返回true. 

如果tryRelease返回true, 判断当前head的状态, 然后执行unparkSuccessor方法. 

```java
private void unparkSuccessor(Node node) { // node == head
    /*
        * If status is negative (i.e., possibly needing signal) try
        * to clear in anticipation of signalling.  It is OK if this
        * fails or if status is changed by waiting thread.
        */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0); // 设置waitStatus为0

    /*
        * Thread to unpark is held in successor, which is normally
        * just the next node.  But if cancelled or apparently null,
        * traverse backwards from tail to find the actual
        * non-cancelled successor.
        */
    Node s = node.next; // 如果node的下个节点为空, 或者s的waitStatus 是cancelled 
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev) // 从队尾开始找可被unpark的Node
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
如果当前释放锁的节点waitStatus < 0, 则重置其状态为0(初始状态). 然后拿到当前节点的下一个节点, 如果这个节点为null或者被cancel了, 则沿着队尾(可能等待队列断链了)找到最后一个等待节点并把这个节点包含的线程唤醒. 释放锁包含释放锁, 唤醒下一个等待的线程.

## ReentrantReadWriteLock部分

ReentrantReadWriteLock也分公平和非公平模式, 我们主要分析非公平锁的逻辑. 

先看读锁lock的过程: 

```java
public void lock() {
    sync.acquireShared(1);
}

//AQS
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
//AQS
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 如果当前已有独占锁(即写锁), 而且获取锁的线程不是当前的线程, 返回-1. 外边执行doAcquireShared, 走排队逻辑
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 获取读锁的获取数
    int r = sharedCount(c);
    // 如果条件符合则直接cas获取读锁
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    // 如果上述没有获取成功, 则spin等待队列获取锁.
    return fullTryAcquireShared(current);
}

// 检查当前排队的队头是否是写锁, 如果是的话给写锁优先权.
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}
```

通过exclusiveCount方法可以获取当前独占锁(即写锁)的数量. 如果当前已经有写锁, 而且锁定的线程不是当前线程, 直接返回失败. 对于读写锁, 如果当前有线程占有写锁, 则读锁不能被获取. 

跳到下一个if, 这有三个判断条件: 

- 这个方法判断当前的获取读锁是否应该被block. 啥时候需要被block? 如果当前等待队列的头节点将要获取写锁, 那么这种情况下读锁需要把优先权让给它(只有这一种情况会, 比如写锁在队列中则会照常按序等待). 
- 读锁的数量是否超了最大锁数限制65535.
- CAS state是否成功

如果三个条件同时为真, 则读锁获取所成功. 否则调用fullTryAcquireShared, 通过自旋来获取锁. 

如果仍然没有获取成功(比如, 写锁一直被占有, 或者等待队列头节点正在等待获取写锁), 则会执行doAcquireShared方法. 

```java
//AQS
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

这段代码我们比较熟悉, 首先把当前线程加到等待队列中. 如果当前节点的前一个节点是head, 则尝试CAS获取锁. 和重入锁不同的是获取成功之后执行的操作: set head 且 把等待的读锁都唤醒.

unlock的过程, 还有写锁几乎和ReentrantLock一致的, 所以这里就不再次分析了. 

## CountDownLatch 部分

CountDownLatch也就是我们常说的闭锁, 它可以实现这样的功能: 等待所有的线程到达. 比如我们把任务分配给了多个线程去执行, 等待所有的线程执行完汇总结果. 类似这种通知机制我们可以用闭锁实现. 实现闭锁的时候, 我们需要指定等待数, 也可以指定所有线程到达时触发的回调函数. 闭锁的两个核心方法是:

- countDown: 通知线程已就绪.
- await: 等待所有的线程到达.

接下来我们分析下闭锁的实现. 首先从它的构造函数看起(说是看它, 其实为了看Sync): 
```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

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
}
```

CountDownLatch在构造的时候新建了一个Sync实例, 构造函数指定的闭锁数量最终通过setState赋值给了AQS的state. 接着我们分析countDown方法():

```java
public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
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

逻辑比较简单, countDown一次, state减少一次. 当state=0(所有线程到达), 执行doReleaseShared方法. 我们再来看下await方法:

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```

当state > 0时, 主线程执行doAcquireSharedInterruptibly方法(会检查线程是否中断), 首先将当前线程添加到等待队列中, 然后线程被中断. 当最后一个线程执行到countDown时, 通过doReleaseShared方法把所有等待线程唤醒.

至此, AQS及其实现类都分析完毕. 文章中只是选择性的分析了一些场景, 并没有面面俱到. 所以, 如果想要更深入全面的了解, 还需自己去看代码, 以及Doug Lea大神的论文, 幸运的是ifeve已经有了中译[版本](http://ifeve.com/aqs/).

## 拜神

![Doug Lea](../images/doug_lea.jpg)

上述我们所看的源码, 都出自Doug Lea之手, 针对AQS的设计还有一片[论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf), 大家可以找来看看.
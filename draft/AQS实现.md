# Head

# Head Head

AQS实现了最基本的同步机制, 是Java中构建锁, 信号量, 以及其它同步工具的基石. AQS内部维护了一个先进先出的队列, head表示当前获取成功的线程, tails表示获取锁失败正在等待的线程. AQS中有一个int 类型的变量: state. 这个属性可以被子类扩展. 在ReentrantLock中state表示获取锁的数量, state=0表示锁未被获取过. 

队列节点有两个属性需要知悉一下: thread和waitStatus. Thread记录当前获取锁的线程. waitStatus表示当前线程等待状态: 

- CANCELLED => 1     节点已取消
- SIGNAL => -1       节点等待触发
- CONDITION => -2    节点等待条件
- PROPAGATE => -3    节点状态需要向后传播

下边分析下ReentrantLock获取非公平锁的过程: 

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

public final void acquire(int arg) { // arg = 1
    // 如果获取锁失败 && 并且当前线程被中断当前线程
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 调用线程的interupt方法.
        selfInterrupt();
}

// 这个方法由不同的子类进行实现
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

// 非公平锁的tryAcquire方法调用了这个方法
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

// 执行自旋, 尝试让等待中的节点获取锁
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
            // 检查当前线程是否应该被中断 && 调用LockSupport.park成功则当前线程可中断.
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) // 如果前一个节点的state是signal, 则当前线程应该interrupt
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

unlock过程
```
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) { // 如果当前线程完全释放锁
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

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
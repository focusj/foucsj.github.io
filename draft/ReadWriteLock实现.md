1. 底层共用了一个AQS实例. ReadLock和WriteLock的构造函数中取了当前锁中的sync变量.
aqs中state类型被分成了两个来用, 高16位用来表示读锁, 低16位用来表示写锁.
```
private final ReentrantReadWriteLock.ReadLock readerLock;
private final ReentrantReadWriteLock.WriteLock writerLock;
final Sync sync;

public ReentrantReadWriteLock() {
    this(false);
}

public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

protected ReadLock(ReentrantReadWriteLock lock) {
    sync = lock.sync;
}

protected WriteLock(ReentrantReadWriteLock lock) {
    sync = lock.sync;
}
```

读锁加锁的过程:
```
public void lock() {
    sync.acquireShared(1);
}

public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

protected final int tryAcquireShared(int unused) {
    /*
        * Walkthrough:
        * 1. If write lock held by another thread, fail.
        * 2. Otherwise, this thread is eligible for
        *    lock wrt state, so ask if it should block
        *    because of queue policy. If not, try
        *    to grant by CASing state and updating count.
        *    Note that step does not check for reentrant
        *    acquires, which is postponed to full version
        *    to avoid having to check hold count in
        *    the more typical non-reentrant case.
        * 3. If step 2 fails either because thread
        *    apparently not eligible or CAS fails or count
        *    saturated, chain to version with full retry loop.
        */
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
    /* As a heuristic to avoid indefinite writer starvation,
        * block if the thread that momentarily appears to be head
        * of queue, if one exists, is a waiting writer.  This is
        * only a probabilistic effect since a new reader will not
        * block if there is a waiting writer behind other enabled
        * readers that have not yet drained from the queue.
        */
    return apparentlyFirstQueuedIsExclusive();
}
```

写锁加锁的过程, 只列出核心方法:
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected final boolean tryAcquire(int acquires) {
    /*
        * Walkthrough:
        * 1. If read count nonzero or write count nonzero
        *    and owner is a different thread, fail.
        * 2. If count would saturate, fail. (This can only
        *    happen if count is already nonzero.)
        * 3. Otherwise, this thread is eligible for lock if
        *    it is either a reentrant acquire or
        *    queue policy allows it. If so, update state
        *    and set owner.
        */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // 如果当前已经有读或写锁, 并且当前线程和获取锁的线程不是同一个抛错.
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 如果超过最大锁数量, 抛异常.
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // 获取可重入锁, 返回成功状态
        setState(c + acquires);
        return true;
    }
    //writerShouldBlock 固定返回false, 因为写锁不能被驳回
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // cas成功, 成功获取锁
    setExclusiveOwnerThread(current);
    return true;
}

acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 这部分的逻辑和上篇文章分析可重入锁的逻辑是几乎一致的.
```
# Netty NioEventLoop

## Reactor 模型

Netty实现并扩展了Reactor模型，为了更好的了解EventLoop，我们有必要先看一下Reactor模型的定义。

![Reactor-Model](../images/Reactor.png)

在wiki对[reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern)的定义中，指出了以下几种角色：

- Resource：资源指的是提供系统输入或者消费系统输出的资源。在Netty中它指的是SocketChannel，它们应支持select。
- Demultiplexer：事件分离器负责对资源进行轮寻等待，当资源ready的时候，分离器负责将数据发送给Dispatcher。
- Dispatcher：处理Handler的注册和反注册。当资源到达时负责把资源分发到相应的Handler中。
- Handler：负责处理数据。

在Netty中EventLoop兼负了Demultiplexer以及Dispatcher两个角色。下边我们通过来看NioEventLoop的源码学习学习并了解Netty中的EventLoop。

## EventLoop源码

NioEventLoop的核心方法是run()方法，一旦Netty程序启动之后，这个就一直循环跑下去，不间断的查询IO和处理task。

```java
protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));

                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                    default:
                        // fallthrough
                }
        }
        ///
}
```

### 关于selectionStrategy

首先我们来看第一个逻辑Select Strategy。这段逻辑主要控制这次循环是执行：跳过；select操作；还是fall through。判断依据是这样的：

```
public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
    return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
}
```

如果当前EventLoop中有未处理的task，则执行selectorNowSupplier。selectorNowSupplier调用了selectNow。selectNow调用的是Selector的selectNow这个非阻塞方法。执行完selectNow则跳出switch运行下边的processSelectedKeys逻辑。

为了高效的利用CPU，EventLoop中只要有未消费的task则优先消费task。

Nio中Selector.select()是阻塞的，直到某个selection key可用select方法才会返回。Selector.selectNow()则检查自从上次select到现在有没有可用的selection key，然后立即返回。

```
private final IntSupplier selectNowSupplier = new IntSupplier() {
    @Override
    public int get() throws Exception {
        return selectNow();
    }
};

int selectNow() throws IOException {
    try {
        return selector.selectNow();
    } finally {
        // restore wakeup state if needed
        if (wakenUp.get()) {
            selector.wakeup();
        }
    }
}

```

### select操作

select操作主要是检查当前的selection key，看哪些已available。

上边我们说到了Selector.select操作是阻塞的，那么如果我不想等了，可以中断它吗？可以，Selector.wakeup可以唤醒正在阻塞的select()操作。但是如果当前没有select操作，执行了wakeUp操作，那么下次执行的select()或者selectNow()操作将被立即唤醒。

但是Selector.wakeup是开销比较大的操作,不能每次都直接调用wakeup，于是NioEventLoop中声明了wakenUp(AtomicBoolean)字段，用于控制selector.wakeup()的调用。调用wakeup之前先`wakenUp.compareAndSet(false, true)`，如果set成功才执行Selector.wakeup()操作。

当用户提交新的任务时executor.execute(...)，会触发wakeup操作。

```
select(wakenUp.getAndSet(false));

if (wakenUp.get()) {
    selector.wakeup();
}
```

这段代码有一段非常长的注释，解释了为什么这段逻辑这样实现。并且给出了什么情况下会产生竞态条件：

```
wakenUp.set(false)
selector.select(...)
```

wakenUp.set(false)执行后，用户出发了wakeup操作，然后执行select操作，这时select将立即返回。直到下次循环把wakenUp重置为false，期间所有的wakenUp.compareAndSet(false, true)都是执行失败的，因为现在wakenUp的值是true。所以接下来的select()都不能被wakeup。

### select 内部逻辑
接下来我们看select是如何实现的：
```
private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
            for (;;) {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if (timeoutMillis <= 0) { // 1
                    if (selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                if (hasTasks() && wakenUp.compareAndSet(false, true)) { // 2
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                int selectedKeys = selector.select(timeoutMillis);
                selectCnt ++;

                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) { // 3
                    break;
                }

                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    logger.warn(
                            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                            selectCnt, selector);

                    // 重建Selector，旧的Selector中的Selection Key要拷贝到新的Selector中
                    rebuildSelector();
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;
            }
        ///    
    }

```

selectCnt标记select执行的次数，用于检测NIO的epoll bug。在这个方法尾部有一个判断：

```
 if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {}
```

判断select记次是否超过了伐值，如果是的话有可能触发了Nio epoll bug，执行重建selector的逻辑：新建一个Selector，把原来老的selection key都复制过去。重建完成之后再执行一次selectNow。

因为select操作是阻塞的，如果长时间没有IO可用，就会造成NioEventLoop中的task积压。因此每次执行select操作都设定一个超时：
1.查询定时任务重最近要被执行的task还有多长时间执行.
2.这个时间加上0.5ms就是最大超时时间。

```
long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
```

整体来看一下这个for循环：

- 第1个if：如果timeoutMillis小于0，则立即执行一次异步的selectNow，跳出循环消费task。
- 第2个if：如果当前taskQueue中有task，并且没有被wakeup，则执行一次异步的selectNow，跳出循环消费task。
- 接下来执行select，并记次。
- 第3个if：如果有available keys 或者 被用户唤醒 或者 任务队列定时队列有任务则中断。
- 最后就是重建selector的过程。

### processSelectedKeys

```
cancelledKeys = 0;
needsToSelectAgain = false;
final int ioRatio = this.ioRatio;
if (ioRatio == 100) {
    try {
        processSelectedKeys();
    } finally {
        runAllTasks();
    }
} else {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys();
    } finally {
        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```

NioEventLoop.run方法的后半段逻辑主要是processSelectedKeys（处理IO）和runTasks（消费任务）。这里有一个参数用于控制处理这两种任务的时间配比：ioRatio。

先来看一下processSelectedKeys，它的逻辑由processSelectedKeysOptimized和processSelectedKeysPlain实现，调用那个函数取决于你是否开启了`DISABLE_KEYSET_OPTIMIZATION`。如果开启了Selection 优化选项，则在创建Selector的时候以反射的方式把SelectedSelectionKeySet selectedKeys设置到selector中。具体实现在openSelector中，代码就不贴出来了。SelectedSelectionKeySet内部是基于Array实现的，而Selector内部selectedKeys是Set类型的，遍历效率Array效率更好一些。

我们来分析processSelectedKeysPlain方法：

```
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
    if (selectedKeys.isEmpty()) {
        return;
    }

    Iterator<SelectionKey> i = selectedKeys.iterator();
    for (;;) {
        final SelectionKey k = i.next();
        final Object a = k.attachment();
        i.remove();

        // 处理channel中的数据
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (!i.hasNext()) {
            break;
        }

        if (needsToSelectAgain) {
            selectAgain();
            selectedKeys = selector.selectedKeys();

            // Create the iterator again to avoid ConcurrentModificationException
            if (selectedKeys.isEmpty()) {
                break;
            } else {
                i = selectedKeys.iterator();
            }
        }
    }
}
```

SelectionKey上边可以挂载Attachment，一般情况下新的链接对象Channel会挂到attachment上。我们在遍历selectedKeys时，首先取出selection key上的attachment，key的类型可能是AbstractNioChannel和NioTask。根据不同的类型调用不同的处理函数。我们着重看处理channel的逻辑：

1.如果selection key是：SelectionKey.OP_CONNECT，那表明这是一个连接操作。对于连接操作，我们需要把这个selection key从intrestOps中清除掉，否则下次select操作会直接返回。接下来调用finishConnect方法。

2.如果selection key是：SelectionKey.OP_WRITE。则执行flush操作，把数据刷到客户端。

3.如果是read操作则调用unsafe.read()。这个操作就不展开了，等到接下来的文章，专门分析read操作。

```
    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        try {
            int readyOps = k.readyOps();
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }

            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }

```

整体来看NioEventLoop的实现也不复杂，主要就干了两件事情：select IO以及消费task。因为select操作是阻塞的（尽管设置了超时时间），每次执行select时都会检查是否有新的task，有则优先执行task。这么做也是做大限度的提高EventLoop的吞吐量，减少阻塞时间。

除了这两件事儿呢，NioEventLoop还解决了JDK中注明的EPoll bug。到此NioEventLoop源码分析完结。
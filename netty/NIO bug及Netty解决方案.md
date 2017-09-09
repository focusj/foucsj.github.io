# NIO Bug及Netty解决方案

这个[Bug](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6481709)本不该再拿出来说了，因为现在新版本的JDK中已经解决了这个问题。不过这个问题太经典了，影响范围太大了，早期版本的JDK谁用NIO谁掉坑。好多NIO框架都被逼的想各自的辙规避。所以还是落井下石一把吧。

下边分析下这个bug产生的原因
```Java
	while (true) {
	    try {
		// select函数是一个阻塞函数，它会阻塞当前线程直到有数据可读。但是在Linux操作系统上，这个函数直接返回了0。这时，下边的
        // while 逻辑根本也不会执行，程序一直执行while语句造成CPU 100% 。
		int numKeys = selector.select();

        if (numKeys == 0) {
            System.err.println("select wakes up with zero!!!");
        }

		Iterator it = selector.selectedKeys().iterator();
		while (it.hasNext()) {
		    SelectionKey selected = (SelectionKey) it.next();
		    int ops = selected.interestOps();

                    try {
                        // process new connection
                        if ((ops & SelectionKey.OP_ACCEPT) != 0) {
                            clientChannel = serverChannel.accept();
                            clientChannel.configureBlocking(false);

                            // register channel to selector
                            clientChannel.register(selector, SelectionKey.OP_READ, null);
                            System.out.println("2. SERVER ACCEPTED AND REGISTER READ OP : client - " + clientChannel.socket().getInetAddress());
                        }

                        if ((ops & SelectionKey.OP_READ) != 0) {
                            // read client message
                            System.out.println("3. SERVER READ DATA FROM client - " + clientChannel.socket().getInetAddress());
                            readClient((SocketChannel) selected.channel(), buffer);

                            // deregister OP_READ
                            System.out.println("PREV SET : " + selected.interestOps());
                            selected.interestOps(selected.interestOps() & ~SelectionKey.OP_READ);
                            System.out.println("NEW SET : " + selected.interestOps());

                            Thread.sleep(SLEEP_PERIOD * 2);
                            new WriterThread(clientChannel).start();
                        }

                    } finally {
                        // remove from selected key set
                        it.remove();
                    }
		}
	    } catch (IOException e) {
		    System.err.println("IO Error : " + e.getMessage());
	    }
	}
    }
```

下边看一下Netty中的解决方案。整体思路来说比较简单：记录loop空转的次数，如果超过指定的伐值则重建SelectionKey。代码中汉语注解部分指出了对bug的处理逻辑。

```java
private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
            for (;;) {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if (timeoutMillis <= 0) {
                    if (selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
                // Selector#wakeup. So we need to check task queue again before executing select operation.
                // If we don't, the task might be pended until select operation was timed out.
                // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                int selectedKeys = selector.select(timeoutMillis);
                // select完成增加select记次
                selectCnt ++;

                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                    // - Selected something,
                    // - waken up by user, or
                    // - the task queue has a pending task.
                    // - a scheduled task is ready for processing
                    break;
                }
                if (Thread.interrupted()) {
                    // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                    // As this is most likely a bug in the handler of the user or it's client library we will
                    // also log it.
                    //
                    // See https://github.com/netty/netty/issues/2426
                    if (logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely because " +
                                "Thread.currentThread().interrupt() was called. Use " +
                                "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                    }
                    selectCnt = 1;
                    break;
                }

                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;

                // 判断select记次是否超过了伐值，如果是的话有可能触发了nio epoll bug，执行重建selector的逻辑。
                // 新建Selector会把原来老的selection key都复制过去。
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    // The selector returned prematurely many times in a row.
                    // Rebuild the selector to work around the problem.
                    logger.warn(
                            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                            selectCnt, selector);

                    rebuildSelector();
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;
            }

            if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
            }
        } catch (CancelledKeyException e) {
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
            // Harmless exception - log anyway
        }
    }
```
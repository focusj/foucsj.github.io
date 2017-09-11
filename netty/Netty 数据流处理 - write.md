# Netty Data Stream Handling - write

[上篇文章](http://www.jianshu.com/p/9e7f8ece92dc)中介绍了Netty是读数据的流程：EventLoop不停的select IO；一旦发现OP_READ可用则利用Channel.Unsafe读取数据，并把数据传给Pipeline；Pipeline拿到数据，并交由内部的Handler处理。还有上篇文章也说了Pipeline中的两种数据流向：inbound和outbound。read操作符合inbound流向；而write则符合outbound刘翔。这些知识对我们理解write操作大有裨益，因为write操作在Pipeline中是read的反向操作。

那我们开始介绍Netty的write写数据流程。先找到EchoServerHandler的channelRead方法，在这个示例中，它会把读到的数据再写回客户端：
```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ctx.write(msg);
}
```

这个方法直接调用了ChannelHandlerContext的write方法：
```
@Override
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}
```
没什么好解释的，给出write的调用链：
```
@Override
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
    if (msg == null) {
        throw new NullPointerException("msg");
    }

    try {
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return promise;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }
    write(msg, false, promise);

    return promise;
}

private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}

private void invokeWrite(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
    } else {
        write(msg, promise);
    }
}

private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```

invokeWrite0接下来会调用HeadContext中的write方法（没什么好解释的了，read的反向操作）：

```
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```

这个方法调用了Unsafe的write方法，至此write操作从Pipeline中走完了，接下来才是重头戏。我们来看这个AbstractUnsafe实现的write方法：

```
@Override
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    // 1
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        // 2
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    // 3
    outboundBuffer.addMessage(msg, size, promise);
}
```
直接调用write写数据的时候，并不是直接写到channel中，而是先写到缓冲区里，也就是ChannelOutboundBuffer。当调用调用flushflush才开始向channel写数据。ChannelOutboundBuffer是一个无界链表，如果不停的向缓冲区写入数据可能会导致内存溢出。因此ChannelOutboundBuffer检测当`newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()`时，buffer便不可写。同时通知Pipeline Writablity Changed：`pipeline.fireChannelWritabilityChanged();`。我们在实现自己的Handler时可以重新实现该方法。

我们接着看下filterOutboundMessage这个方法：
```
@Override
    protected final Object filterOutboundMessage(Object msg) {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            if (buf.isDirect()) {
                return msg;
            }

            return newDirectBuffer(buf);
        }

        if (msg instanceof FileRegion) {
            return msg;
        }

        throw new UnsupportedOperationException(
                "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
    }
```

在讲解ByteBuf的[文章](http://www.jianshu.com/p/20329efe3ca4)中提到过关于使用直接内存还是堆内存的最佳实践：*在IO通信的线程中操作ByteBuf应使用DirectBuffer(省去内存拷贝的成本)，在后端业务逻辑中操作ByteBuf应使用HeapBuffer(不用担心内存泄露)*。

这个方法首先检查msg类型是不是ByteBuf，如果是ByteBuf则检查是不是使用了直接内存，如果没有则把基于堆的内存换成直接内存。

我们接着分析ChannelOutboundBuffer的addMessage方法：
```
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```
这个方法做了两件事情：把msg加入到链表中；调用incrementPendingOutboundBytes方法。我们来看这个方法：
```
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }

    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);

    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        setUnwritable(invokeLater);
    }
}
```
首先，先更新缓冲区大小，接着判断缓冲区是否大于最高水位，如果是则设置buffer为unWritable（默认的高水位线大小是64K）。setUnwritable方法内通知了Pipeline Writablity Changed，这里不贴代码了自己看去吧。

到这的话write方法是执行完了，但是数据仍然留在内存中。接下来我们看flush方法。跳过Pipeline直接来看AbstractUnsafe的flush：

```
@Override
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }

    outboundBuffer.addFlush();
    flush0();
}
```

这个方法首先调用了outboundBuffer的addFlush方法，我们先看下addFlush做了什么：

```
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        unflushedEntry = null;
    }
}
```

unflushedEntry表示缓冲区链表中第一个未被flush的元素。如果这个变量为null的话，表示当前缓冲区已被flush。for循环中首先把entry的promise设置为Uncancellable（Promise继承自Future，Future是可以cancel的），然后增加flushed计数。decrementPendingOutboundBytes方法中有逻辑检查目前Buffer是否低于低水位，如果是则重置Buffer为可写。

我们接着看flush0方法，flush0主要是调用NioSocketChannel的doWrite方法：

```
protected void flush0() {
    ///...
    doWrite(outboundBuffer);
    ///...
}

protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    for (;;) {
        int size = in.size();
        if (size == 0) {
            clearOpWrite();
            break;
        }
        long writtenBytes = 0;
        boolean done = false;
        boolean setOpWrite = false;

        // 1
        ByteBuffer[] nioBuffers = in.nioBuffers();
        int nioBufferCnt = in.nioBufferCount();
        long expectedWrittenBytes = in.nioBufferSize();
        SocketChannel ch = javaChannel();

        switch (nioBufferCnt) {
            case 0:
                super.doWrite(in);
                return;
            case 1:
                ByteBuffer nioBuffer = nioBuffers[0];
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    // 2
                    final int localWrittenBytes = ch.write(nioBuffer);
                    if (localWrittenBytes == 0) {
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) {
                        done = true;
                        break;
                    }
                }
                break;
            default:
                ///...
        }

        in.removeBytes(writtenBytes);

        if (!done) {
            // Did not write all buffers completely.
            incompleteWrite(setOpWrite);
            break;
        }
    }
}
```

doWrite方法首先获取Buffer中缓存的消息，并将Netty ByteBuf转成Nio ByteBuffer；然后把数据写入Nio SocketChannel。写数据的操作放到了一个for循环中。因为Netty的一个缓冲数据可能不会一次性的刷到Channel中，如果只作一次write操作就返回，那么很有可能余下的数据要等到下次OP_WRITE Selection Key可用才能全部写完。这期间经历了一次昂贵的IO操作，所以这个for循环又是CPU时间和IO时间的一个精心分配。

至此，Netty写数据的流程分析完毕。
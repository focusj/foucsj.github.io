# Netty源码走读-ServerBootStrap启动过程 (二)

## ChannelPipeline工作流程 

问题：问什么使用Unsafe屏蔽Channel的底层操作？
（为什么使用unsafe，unsafe的实现在不同的channel实现中，然后unsafe调用其实现，实现不同channel的物理处理，read，bind等等）
(比较channel和unsafe的api有何异同)


在上篇文章中我们看过AbstractChannel类的构造函数，其中有一行代码是pipeline的初始化:

```Java
    protected AbstractChannel(Channel parent, ChannelId id) {
        this.parent = parent;
        this.id = id;
        unsafe = newUnsafe();
        pipeline = newChannelPipeline(); // <-
    }

    protected DefaultChannelPipeline newChannelPipeline() {
        return new DefaultChannelPipeline(this);
    }
```

在channel register的过程中从SingleThreadEventLoop调用unsafe的register方法。unsafe中调用pipeline上的方法。这是合理的，因为unsafe主要是底层操作的门面，所以当unsafe拿到事件后触发Pipeline。

所以


ChannelHandler定义了两个最重要的子接口ChannelInboundHandler和ChannelOutboundHandler。
ChannelInboundHandler当有数据流入或者与其对应的Channel状态发生改变时会调用。
ChannelOutboundHandler当有数据流出或类似的操作发生时调用。

由于bind流程需要了解一些pipeline的知识，所以先开一个小差，大概的介绍一下Pipeline，然后再来分析bind流程。每个Channel在初始化的时候都会关联一个pipiline。ChannelPipeline是Netty中处理事件的管线。一个Pipeline可以注册多个事件处理器ChannelHandler，当有事件进入Pipiline中会分别由其中注册的ChannelHandler处理。下图是ChannelPipeline的设计图。流经Pipeline的事件，根据其流向可以分为bottom-up，top-down。ChannelHandler根据其能处理的不同流向的事件分为ChannelInboundHandler和ChannelOutboundHandler。bottom-up流向的事件由InboundHandler处理，top-down流向的事件由OutboundHandler处理。

```
                                                 I/O Request
                                            via Channel or
                                        ChannelHandlerContext
                                                      |
  +---------------------------------------------------+---------------+
  |                           ChannelPipeline         |               |
  |                                                  \|/              |
  |    +---------------------+            +-----------+----------+    |
  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  .               |
  |               .                                   .               |
  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
  |        [ method call]                       [method call]         |
  |               .                                   .               |
  |               .                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  +---------------+-----------------------------------+---------------+
                  |                                  \|/
  +---------------+-----------------------------------+---------------+
  |               |                                   |               |
  |       [ Socket.read() ]                    [ Socket.write() ]     |
  |                                                                   |
  |  Netty Internal I/O Threads (Transport Implementation)            |
  +-------------------------------------------------------------------+
```

举个例子：
```
 ChannelPipeline p = ...;
 p.addLast("1", new InboundHandlerA());
 p.addLast("2", new InboundHandlerB());
 p.addLast("3", new OutboundHandlerA());
 p.addLast("4", new OutboundHandlerB());
 p.addLast("5", new InboundOutboundHandlerX());
```
初始化一个pipeline，里边注册了5个ChannelHandler。当产生bottom-up的事件时，处理事件的Handler分别是：1 -> 2 -> 5；当产生top-down的事件时，处理事件的Handler分别是：5 -> 4 -> 3。



1. Channel的四种状态。
ChannelRegister(Channel注册到EventLoop) -> ChannelActive(已链接到远程，可以读写数据) -> ChannelInactive(断开，不能读写数据) -> ChannelUnRegister(未注册到Event)




接下来我们又得回到AbstractBootStrap的doBind方法，来看doBind0的实现：

```Java
    private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

这个方法里调用了Channel的bind方法，上文中我们已经知道了bind方法是Channel提供的，还知道Channel方法是一个接口。所哟，我们直接去它的子类里找bind的实现就好了。AbstractChannel就发现了bind的定义：

```Java
    public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
        return pipeline.bind(localAddress, promise);
    }
```

bind其实是调用pipeline的bind()。如果足够细心的话你会发现，pipeline是在AbstractChannel的构造函数中初始化的，我们来扒一下Pipeline是如何创建的：

```Java
    protected DefaultChannelPipeline newChannelPipeline() {
        return new DefaultChannelPipeline(this);
    }
```

非常简单，创建了一个DefaultChannelPipeline实例。我们接着看bind逻辑，DefaultChannelPipeline的bind方法是这样实现的：

```Java
    public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
        return tail.bind(localAddress, promise);
    }
```

我们继续跟进bind方法，进入AbstractCahnnelHandlerContext中：
```
    @Override
    public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        if (isNotValidPromise(promise, false)) {
            // cancelled
            return promise;
        }

        final AbstractChannelHandlerContext next = findContextOutbound();
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeBind(localAddress, promise);
        } else {
            safeExecute(executor, new Runnable() {
                @Override
                public void run() {
                    next.invokeBind(localAddress, promise);
                }
            }, promise, null);
        }
        return promise;
    }

    private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
        if (invokeHandler()) {
            try {
                ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
            } catch (Throwable t) {
                notifyOutboundHandlerException(t, promise);
            }
        } else {
            bind(localAddress, promise);
        }
    }
```



# Vert.x

## 

Fluent Api.

Multi-Reactor

线程模型, 与Netty一脉相承, EventLoop.



应用组织借鉴自Actor模型, Verticle.

EventBus. EventBus是Verticle之间通信的唯一渠道.

集群的支持. 


Vertx


```java
  public void runOnContext(Handler<Void> task) {
    ContextImpl context = getOrCreateContext();
    context.runOnContext(task);
  }
```
查询或者创建一个新的Context. 
```java
  public ContextImpl getOrCreateContext() {
    ContextImpl ctx = getContext();
    if (ctx == null) {
      // We are running embedded - Create a context
      ctx = createEventLoopContext(null, null, new JsonObject(), Thread.currentThread().getContextClassLoader());
    }
    return ctx;
  }

  public ContextImpl getContext() {
    ContextImpl context = (ContextImpl) context();
    if (context != null && context.owner == this) {
      return context;
    }
    return null;
  }

  public static Context context() {
    Thread current = Thread.currentThread();
    if (current instanceof VertxThread) {
      return ((VertxThread) current).getContext();
    }
    return null;
  }
```

获取Context的逻辑: 判断当前线程是不是VertxThread, 如果是则拿到VertxThread上绑定的线程, 如果不是则创建一个新的EventLoopContext. 

Context主要有三个继承类: EventLoopContext, WorkerContext, MultiThreadedWorkerContext. 

这是ContextImpl的构造函数: 
```java
protected ContextImpl(VertxInternal vertx, WorkerPool internalBlockingPool, WorkerPool workerPool, String deploymentID, JsonObject config,
                        ClassLoader tccl) {
    if (DISABLE_TCCL && !tccl.getClass().getName().equals("sun.misc.Launcher$AppClassLoader")) {
      log.warn("You have disabled TCCL checks but you have a custom TCCL to set.");
    }
    this.deploymentID = deploymentID;
    this.config = config;
    EventLoopGroup group = vertx.getEventLoopGroup();
    if (group != null) {
      this.eventLoop = group.next();
    } else {
      this.eventLoop = null;
    }
    this.tccl = tccl;
    this.owner = vertx;
    this.workerPool = workerPool;
    this.internalBlockingPool = internalBlockingPool;
    this.orderedTasks = new TaskQueue();
    this.internalOrderedTasks = new TaskQueue();
    this.closeHooks = new CloseHooks(log);
  }
```
在创建Context实例的时候已经和eventloop绑定了, 首先拿到vertx上EventLoopGroup然后取出一个EventLoop绑定到Context上. TaskQueue是用于Woker暂时存放任务用. ClassLoader是取的当前线程的ContextClassLoader.

然后看runOnContext方法: 

```java
  public void runOnContext(Handler<Void> task) {
    try {
      executeAsync(task);
    } catch (RejectedExecutionException ignore) {
      // Pool is already shut down
    }
  }
```

 executeAsync是一个抽象方法，EventLoopContext，WorkerContext，MultiThreadedContext各自有不同的实现。
 EventLoopContext：把task会被包装成Runnable 提交到Netty的EventLoop中执行。对于EventLoop是如何处理Task的这个我们已经分析过了. 
 WorkerContext/MultiThreadedContext：把task提交到orderedTaskQueue中。

 ```java
   public void executeAsync(Handler<Void> task) {
    nettyEventLoop().execute(wrapTask(null, task, true, null));
  }
 ```
## 背景

最近，在公司主要负责实时业务监控系统的开发。系统大概分为三大块：

1. 应用SDK。对应用方我们封装了SDK，提供counter，guage，histogram这三种接口。SDK内置一个定时任务，定期的把指标数据发送到Data Server。

2. 服务网关（Data Server）。网关会对链接进行鉴权和限流；对于应用上报的数据，转化成JSON串丢到Kafka。
   
3. 对报警数据的清洗，存储、分析等操作都由不同的Kafka Consumer负责。

整套系统使用Java开发，用到了Spring Boot和gRPC框架。目前我们共有三个Consumer，作用分别是：数据写入InfluxDB（数据展示使用的是Grafana）；抽取指标元数据信息并写入MySQL（配置报警策略会用到）；分析报警策略，实时报警。

Consumer和Data Server目前暂时在一个项目中。

## 问题

上线后系统运行的一直稳定，但就是最近常收到一些用户的反馈。有的说Grafana配置的图表中断；还有说曲线走势不正常，某个时间点出现陡将（经检查业务系统运行正常）。

收到问题反馈后，由于内心盲目的自信。之前运行好好的系统，不可能出问题啊（原来确实连续几个月运行都没出现过问题）。登陆服务器象征性的看了下日志，确实是有一堆异常出现。这股自信劲都让我开始怀疑这是框架的bug。

```java
org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: Java heap space
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:982)
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:901)
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
    at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:861)
    at javax.servlet.http.HttpServlet.doHead(HttpServlet.java:245)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:658)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:742)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.catalina.core.ApplicationDispatcher.invoke(ApplicationDispatcher.java:728)
    at org.apache.catalina.core.ApplicationDispatcher.processRequest(ApplicationDispatcher.java:469)
    at org.apache.catalina.core.ApplicationDispatcher.doForward(ApplicationDispatcher.java:392)
    at org.apache.catalina.core.ApplicationDispatcher.forward(ApplicationDispatcher.java:311)
    ... ...
```

内存用光了，是给太小了吧？最近核心业务强制性接入监控，数据量增加，然后引发了OOM（这个想法当然是错误的，后续会分析）。也没多想，把内存从4G调到了8G，以为这样就没事儿了（事情当然不会这么简单）。

好景不长，没过两天问题又出现了，现象和错误还是一样的。脸被打的啪啪响，老实的排查问题吧。登上服务器，发现Java进程cpu使用率飙到了100，内存占用已经都达到了十几G。伴随着日志中出现的OutOfMemory错误，这基本可以确诊内存泄漏了。找到GC log发现连续的抛出：Allocation Error。内存泄漏实锤。（关于cpu的使用率，可以进一步定位是哪个线程造成的，我的case比较明显）

## 排查

先例行来个heap dump：`jmap -dump:format=b,file=heapDump.hprof 0`。因为在docker启动实例，所以jvm进程号为0，如果实体机上跑了多个Java程序，可以使用`jps -l`查找相关的Java进程。

漫长的等待，终于等到jmap执行完成。dump出来的文件有11G，压缩之后还剩5个多G。下载到本地进行内存分析。

由于文件体积巨大，用VisualVM和jhat这两个工具根本没法打开文件。于是换JProfiler，终于可以看到堆的使用情况：

![monitor-mdump](../images/monitor-mdump.png)

看到这个图，心里其实是蒙圈的。两个内存占用大户：char[]，String，它们就在那里，但是你就是不能拿它们怎么样。因为实在想不出为啥程序会生成这么多String。

硬着头皮接着看，过滤掉和应用程序无关的类，只剩下Metric和MetricMetaCollectorService$$Lamabda$18。Metric是上报的指标对应的dto；MetricMetaCollectorService主要做的事情是：抽取指标元数据信息并写入MySQL。忽然想起MetricMetaCollectorService是最近才上线的功能，难道是它引起的？先给它扣个嫌疑犯的帽子，看能不能根据手上的线索定罪。

接着看内存占用情况，这时发现了另外一个疑点：java.util.concurrent.LinkedBlockingQueue$Node。数量上它和Metric几乎一致。但这个类我确实没在业务中直接使用。会不会和线程池有关（因为之前读过线程池相关的源码，所以下意识的有这样的一个猜测）？

接着需要做两件事情：

1. 检查业务中和service相关的逻辑是否用到了线程池。代开代码一番寻觅，发现在向数据库写入数据时使用线程池做了并行处理。

```java
    public void collect(Metric metric) {
        executor.execute(() -> {
            List<MetricMeta> metas = getMetricMeta(metric);
            List<MetricMeta> nonExistsMetas = metas.stream()
                .filter(meta -> !metricMetaRepo.checkExist(meta))
                .collect(Collectors.toList());
            metricMetaRepo.insertBatch(nonExistsMetas);
        });
    }
```

1. 线程池在初始化的时候使用的什么working queue。线程池我用的是Spring提供的，初始化时指定了核心线程数和最大线程数。但是没有指定working queue和discard policy。


```java
    @Bean(name = "internalExecutor")
    public ThreadPoolTaskExecutor getInternalExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        return executor;
    }
```

我们看下Spring源码中是怎么实现的。打开ThreadPoolTaskExecutor文件，找到initializeExecutor方法：

```java
    protected ExecutorService initializeExecutor(
			ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {
        // 创建working queue逻辑
		BlockingQueue<Runnable> queue = createQueue(this.queueCapacity);

		ThreadPoolExecutor executor;
		if (this.taskDecorator != null) {
			executor = new ThreadPoolExecutor(
					this.corePoolSize, this.maxPoolSize, this.keepAliveSeconds, TimeUnit.SECONDS,
					queue, threadFactory, rejectedExecutionHandler) {
				@Override
				public void execute(Runnable command) {
					super.execute(taskDecorator.decorate(command));
				}
			};
		}
		else {
			executor = new ThreadPoolExecutor(
					this.corePoolSize, this.maxPoolSize, this.keepAliveSeconds, TimeUnit.SECONDS,
					queue, threadFactory, rejectedExecutionHandler);

		}
	}
```

createQueue具体实现是这样的：如果指定的queueCapacity > 0 则根据指定的容量创建一个LinkedBlockingQueue。否则创建一个SynchronousQueue。
```java
	protected BlockingQueue<Runnable> createQueue(int queueCapacity) {
		if (queueCapacity > 0) {
			return new LinkedBlockingQueue<Runnable>(queueCapacity);
		}
		else {
			return new SynchronousQueue<Runnable>();
		}
	}
```

queueCapacity的默认值是`Integer.MAX_VALUE`，而我们在创建Executor时也没有指定容量，所以可以确定我们创建的线程池实例确实用到了LinkedBlockingQueue。到这一步基本能够定位问题了：MetricMetaCollectorService异步化，导致了线程池working queue任务堆积导致内存耗尽。

问题的成因总结一下：近期公司强制核心系统接入监控，数据量翻了几番。线程池消费能力不足，导致数据积压撑爆内存。原来在数据量少的时候，线程池可以忙活过来，所以一切都没问题。抑或是已经出现了问题，但是不会这么快的发生。但是数据量一旦增加到一定程度，则催化了这个问题出现。

后续把相应的逻辑删除后，服务恢复正常迄今再未出现过断点。后续会专门有一篇文章专门讲这块逻辑是怎么优化上线的。

## 反思

1. 使用线程池，还请三思。

- 一思：线程池不是银弹，当前的场景是不是非得用线程池才能解决问题。比如我这个场景用了线程池反而给自己挖坑了。
- 二思：思考任务特性。任务特性归类下，可以分成三种：批处理，任务已经固定不会变化；Queue Consumer，任务出现的速率大致固定；API请求，任务出现的速率不稳定，时快时慢。这时理解核心线程数和最大线程数就非常必要了。
- 三思：容错和降级方案。降级：假如线程池处理能力不足，working queue设置多大合适？容错：working queue都满了，如果还有新任务，应该咋整？这时需要配置合理的queue和abort policy。

2. 系统出问题时，不要有侥幸心理，想着重启能够解决。及时查看日志定位问题，不要破坏犯罪现场和第一手证据。如果是内存问题，先heap dump把当前堆的使用情况导出来。如果是cpu问题，可以进一步再确定线程，然后再结合thread dump文件排查问题。

3. 项目启动参数中记着加上gc log和oom heap dump这两个参数。

4. 条件允许把JDK提供的各种分析工具也装上：jps，jstat，jmap。


## 延伸
- [关键业务系统的JVM参数推荐(2018仲夏版)](http://calvin1978.blogcn.com/articles/jvmoption-7.html)
- [JVM性能调优监控工具jps、jstack、jmap、jhat、jstat、hprof使用详解](https://my.oschina.net/feichexia/blog/196575)

# 实战单机百万Tcp连接

## 准备工作 - Server端

1. 进程最大文件打开数，这个参数默认值是很小，如果不调整建立过多的连接时会跑：Too many open files错误。

查看本机的默认最大文件打开数：
```
ulimit -n
```

临时修改方案：
```
ulimit -Sn 1048576
ulimit -Hn 1048576
```
我的ubuntu 17.04在做这个修改的时候遇到一点问题，总是设置不成功，最后发现是系统bug。

永久修改方案，需要修改/etc/security/limits.conf：
```
*         hard    nofile      1048576
*         soft    nofile      1048576
root      hard    nofile      1048576
root      soft    nofile      1048576
```
`*`代表非root用户。修改之后需重启系统或者重新登录使配置生效。

设置完成之后我们可以启动进程检测当前进程的最大文件打开数：`cat /proc/${pic}/limits`:
```
 ~/WorkSpace/Java/vert.x   sc  cat /proc/27136/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             46651                46651                processes 
Max open files            1048576              1048576              files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       46651                46651                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us     
```
可以看到open files一行已经变成我们的预设值。

在修改的时候需要注意一点，soft 不能大于hard值。

2. 系统最大文件打开数：

查看系统最大文件打开数：`cat /proc/sys/fs/file-max`

临时性设置：

`echo 1200000 > /proc/sys/fs/file-max`

永久性性设置：

在/etc/sysctl.conf中添加`fs.file-max = 1200000`，设置完成后执行命令：sudo sysctl -p，重新装载配置。


3. 查看当前系统使用的打开文件描述符数：cat /proc/sys/fs/file-nr。
```
 ~/WorkSpace/Java/vert.x   sc  cat /proc/sys/fs/file-nr 
11069	0	2000000
```
其中第一个数表示当前系统已分配使用的打开文件描述符数，第二个数为分配后已释放的（目前已不再使用），第三个数是系统最大文件数，等于file-max值。


1. 测试现象一：Too many open files。

Exception in thread "vert.x-eventloop-thread-0" java.lang.InternalError: java.io.FileNotFoundException: /usr/lib/jvm/java-8-oracle/jre/lib/ext/cldrdata.jar (Too many open files)
	at sun.misc.URLClassPath$JarLoader.getResource(URLClassPath.java:1040)
	at sun.misc.URLClassPath.getResource(URLClassPath.java:239)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:365)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:362)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:361)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:411)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:335)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	at java.util.ResourceBundle$RBClassLoader.loadClass(ResourceBundle.java:503)
	at java.util.ResourceBundle$Control.newBundle(ResourceBundle.java:2640)
	at java.util.ResourceBundle.loadBundle(ResourceBundle.java:1501)
	at java.util.ResourceBundle.findBundle(ResourceBundle.java:1465)
	at java.util.ResourceBundle.findBundle(ResourceBundle.java:1419)
	at java.util.ResourceBundle.getBundleImpl(ResourceBundle.java:1361)
	at java.util.ResourceBundle.getBundle(ResourceBundle.java:845)
	at java.util.logging.Level.computeLocalizedLevelName(Level.java:265)
	at java.util.logging.Level.getLocalizedLevelName(Level.java:324)
	at java.util.logging.SimpleFormatter.format(SimpleFormatter.java:165)
	at java.util.logging.StreamHandler.publish(StreamHandler.java:211)
	at java.util.logging.ConsoleHandler.publish(ConsoleHandler.java:116)
	at java.util.logging.Logger.log(Logger.java:738)
	at io.netty.util.internal.logging.JdkLogger.log(JdkLogger.java:606)
	at io.netty.util.internal.logging.JdkLogger.warn(JdkLogger.java:482)
	at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:861)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.io.FileNotFoundException: /usr/lib/jvm/java-8-oracle/jre/lib/ext/cldrdata.jar (Too many open files)
	at java.util.zip.ZipFile.open(Native Method)
	at java.util.zip.ZipFile.<init>(ZipFile.java:225)
	at java.util.zip.ZipFile.<init>(ZipFile.java:155)
	at java.util.jar.JarFile.<init>(JarFile.java:166)
	at java.util.jar.JarFile.<init>(JarFile.java:103)
	at sun.misc.URLClassPath$JarLoader.getJarFile(URLClassPath.java:930)
	at sun.misc.URLClassPath$JarLoader.access$800(URLClassPath.java:791)
	at sun.misc.URLClassPath$JarLoader$1.run(URLClassPath.java:876)
	at sun.misc.URLClassPath$JarLoader$1.run(URLClassPath.java:869)
	at java.security.AccessController.doPrivileged(Native Method)
	at sun.misc.URLClassPath$JarLoader.ensureOpen(URLClassPath.java:868)
	at sun.misc.URLClassPath$JarLoader.getResource(URLClassPath.java:1038)
	... 26 more
Aug 21, 2017 12:46:02 PM io.netty.util.concurrent.DefaultPromise safeExecute
SEVERE: Failed to submit a listener notification task. Event loop shut down?
java.util.concurrent.RejectedExecutionException: event executor terminated
	at io.netty.util.concurrent.SingleThreadEventExecutor.reject(SingleThreadEventExecutor.java:821)
	at io.netty.util.concurrent.SingleThreadEventExecutor.offerTask(SingleThreadEventExecutor.java:327)
	at io.netty.util.concurrent.SingleThreadEventExecutor.addTask(SingleThreadEventExecutor.java:320)
	at io.netty.util.concurrent.SingleThreadEventExecutor.execute(SingleThreadEventExecutor.java:746)
	at io.netty.util.concurrent.DefaultPromise.safeExecute(DefaultPromise.java:760)
	at io.netty.util.concurrent.DefaultPromise.notifyListeners(DefaultPromise.java:428)
	at io.netty.util.concurrent.DefaultPromise.setFailure(DefaultPromise.java:113)
	at io.netty.channel.DefaultChannelPromise.setFailure(DefaultChannelPromise.java:87)
	at io.netty.channel.AbstractChannelHandlerContext.safeExecute(AbstractChannelHandlerContext.java:1011)
	at io.netty.channel.AbstractChannelHandlerContext.close(AbstractChannelHandlerContext.java:611)
	at io.netty.channel.AbstractChannelHandlerContext.close(AbstractChannelHandlerContext.java:466)
	at io.netty.channel.DefaultChannelPipeline.close(DefaultChannelPipeline.java:964)
	at io.netty.channel.AbstractChannel.close(AbstractChannel.java:234)
	at io.netty.resolver.dns.DnsNameResolver.close(DnsNameResolver.java:367)
	at io.netty.resolver.dns.InflightNameResolver.close(InflightNameResolver.java:61)
	at io.netty.resolver.InetSocketAddressResolver.close(InetSocketAddressResolver.java:94)
	at io.netty.resolver.AddressResolverGroup$1.operationComplete(AddressResolverGroup.java:79)
	at io.netty.util.concurrent.DefaultPromise.notifyListener0(DefaultPromise.java:507)
	at io.netty.util.concurrent.DefaultPromise.notifyListeners0(DefaultPromise.java:500)
	at io.netty.util.concurrent.DefaultPromise.notifyListenersNow(DefaultPromise.java:479)
	at io.netty.util.concurrent.DefaultPromise.access$000(DefaultPromise.java:34)
	at io.netty.util.concurrent.DefaultPromise$1.run(DefaultPromise.java:431)
	at io.netty.util.concurrent.GlobalEventExecutor$TaskRunner.run(GlobalEventExecutor.java:233)
	at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
	at java.lang.Thread.run(Thread.java:748)

Aug 21, 2017 12:46:02 PM io.netty.util.concurrent.DefaultPromise safeExecute
SEVERE: Failed to submit a listener notification task. Event loop shut down?
java.util.concurrent.RejectedExecutionException: event executor terminated
	at io.netty.util.concurrent.SingleThreadEventExecutor.reject(SingleThreadEventExecutor.java:821)
	at io.netty.util.concurrent.SingleThreadEventExecutor.offerTask(SingleThreadEventExecutor.java:327)
	at io.netty.util.concurrent.SingleThreadEventExecutor.addTask(SingleThreadEventExecutor.java:320)
	at io.netty.util.concurrent.SingleThreadEventExecutor.execute(SingleThreadEventExecutor.java:746)
	at io.netty.util.concurrent.DefaultPromise.safeExecute(DefaultPromise.java:760)
	at io.netty.util.concurrent.DefaultPromise.notifyListeners(DefaultPromise.java:428)
	at io.netty.util.concurrent.DefaultPromise.setFailure(DefaultPromise.java:113)
	at io.netty.channel.DefaultChannelPromise.setFailure(DefaultChannelPromise.java:87)
	at io.netty.channel.AbstractChannelHandlerContext.safeExecute(AbstractChannelHandlerContext.java:1011)
	at io.netty.channel.AbstractChannelHandlerContext.close(AbstractChannelHandlerContext.java:611)
	at io.netty.channel.AbstractChannelHandlerContext.close(AbstractChannelHandlerContext.java:466)
	at io.netty.channel.DefaultChannelPipeline.close(DefaultChannelPipeline.java:964)
	at io.netty.channel.AbstractChannel.close(AbstractChannel.java:234)
	at io.netty.resolver.dns.DnsNameResolver.close(DnsNameResolver.java:367)
	at io.netty.resolver.dns.InflightNameResolver.close(InflightNameResolver.java:61)
	at io.netty.resolver.InetSocketAddressResolver.close(InetSocketAddressResolver.java:94)
	at io.netty.resolver.AddressResolverGroup$1.operationComplete(AddressResolverGroup.java:79)
	at io.netty.util.concurrent.DefaultPromise.notifyListener0(DefaultPromise.java:507)
	at io.netty.util.concurrent.DefaultPromise.notifyListeners0(DefaultPromise.java:500)
	at io.netty.util.concurrent.DefaultPromise.notifyListenersNow(DefaultPromise.java:479)
	at io.netty.util.concurrent.DefaultPromise.access$000(DefaultPromise.java:34)
	at io.netty.util.concurrent.DefaultPromise$1.run(DefaultPromise.java:431)
	at io.netty.util.concurrent.GlobalEventExecutor$TaskRunner.run(GlobalEventExecutor.java:233)
	at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
	at java.lang.Thread.run(Thread.java:748)

查看系统最大文件打开数设置：
cat /proc/sys/fs/file-nr


修改系统值：
sudo vi /etc/sysctl.conf

fs.file-max = 1020000
net.ipv4.ip_conntrack_max = 1020000
net.ipv4.netfilter.ip_conntrack_max = 1020000

解决ubuntu 17.04中总是不能设置成功的问题：https://superuser.com/questions/1200539/cannot-increase-open-file-limit-past-4096-ubuntu



以上做法只是临时，系统下次重启，会还原。 更为稳妥的做法是修改/etc/sysctl.conf文件，增加一行内容

net.ipv4.ip_local_port_range= 1024 65535

每个IP只能创建大概6万多个connection（我本机测试64500个链接）。




# Netty ByteBuf

首先来看下ByteBuf的基本结构。ByteBuf由一段地址空间，一个read index和一个write index组成。两个index分别记录读写进度，省去了NIO中ByteBuffer手动调用flip和clear的烦恼。

```sh
      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      |                   |     (CONTENT)    |                  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity
```

通过上边的图可以很好的理解ByteBuf的数据划分。writer index到capacity之间的部分是空闲区域，可以写入数据；reader index到writer index之间是已经写过还未读取的可读数据；0到reader index是已读过可以释放的区域。

三个index之间的关系是：reader index <= writer index <= capacity

ByteBuf根据其数据存储空间不同有可以分为三种：基于JVM堆内的，基于直接内存的和组合的。堆内受JVM垃圾收集器的管辖，使用上相对安全一些，不用费心内存的释放。弊端是GC是会影响性能的；还有就是内存的拷贝带来的性能损耗。这里拷贝指的是：JVM与内核空间之间的数据复制。

直接内存则不受JVM的管辖，省去了向JVM拷贝数据的麻烦。但是坏处就是别忘了释放内存，否则就会发生内存泄露。

最佳实践：在IO通信的线程中的读写Buffer使用DirectBuffer，在后端业务消息的处理使用HeapBuffer。

通过hasArray检查一个ByteBuf heap based还是direct buffer

ByteBuf提供了两个工具类来创建ByteBuf，分别是支持池化的Pooled和普通的Unpooled。Pooled工具缓存了ByteBuf的实例，提高了性能并且最大限度的减少内存碎片。它使用Jemalloc来搞笑的分配内存。

如果我们有channel的实例还可以通过channel.alloc()来创建ByteBuf实例，它会使用默认的方式创建。

index标记与恢复
markReaderIndex可以对当前的reader index打一个标记，后续如果还有读操作，调用resetReaderIndex可以把readerIndex重置到原来的位置。

空间释放
discardReadByte可以把读过的空间释放，这时buffer的可写空间和writerIndex会相应的改变。discardReadBytes在内存紧张的时候有用，但是调用该方法会伴随buffer的内存整理的。

clear是把readerIndex和writerIndex重置到0，这时writable capacity恢复到原始值。

ByteBuf的派生与复制
派生操作會产生一个新的ByteBuf实例。这里的新指得是byte buf的引用是新的所有的index也是新的。但是它们共用着一套底层存储。派生函数：

- duplicate()
- slice()
- slice(int, int)
- readSlice(int)
- retainedDuplicate()
- retainedSlice()
- retainedSlice(int, int)
- readRetainedSlice(int)

如果想要复制一个ByteBuffer请使用copy，这会完全的复制一个新的ByteBuf出来。

引用计数
引用计数记录了当前ByteBuf被引用的次数。新建一个ByteBuf它的refCnt是1，当refCnt == 0时，这个ByteBuf即可被回收。

查询字符
很多时候需要从ByteBuf中查找特定的字符，比如LineBasedFrameDecoder需要查找'\r\n'出现的地方。ByteBuf提供了简单的indexOf这样的函数。同时也可以使用ByteProcesser来查找。
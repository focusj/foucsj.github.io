# Netty ByteBuf

### ByteBuf的基本结构

ByteBuf由一段地址空间，一个read index和一个write index组成。两个index分别记录读写进度，省去了NIO中ByteBuffer手动调用flip和clear的烦恼。

```sh
      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      |                   |     (CONTENT)    |                  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity
```

通过上图可以很好的理解ByteBuf的数据划分。writer index到capacity之间的部分是空闲区域，可以写入数据；reader index到writer index之间是已经写过还未读取的可读数据；0到reader index是已读过可以释放的区域。

三个index之间的关系是：reader index <= writer index <= capacity

### 存储空间

ByteBuf根据其数据存储空间不同有可以分为三种：基于JVM堆内的，基于直接内存的和组合的。

堆内受JVM垃圾收集器的管辖，使用上相对安全一些，不用每次手动释放。弊端是GC是会影响性能的；还有就是内存的拷贝带来的性能损耗(JVM进程到Socket)。

直接内存则不受JVM的管辖，省去了向JVM拷贝数据的麻烦。但是坏处就是别忘了释放内存，否则就会发生内存泄露。相比于堆内存，直接内存的的分配速度也比较慢。

最佳实践：在IO通信的线程中的读写Buffer使用DirectBuffer(省去内存拷贝的成本)，在后端业务消息的处理使用HeapBuffer(不用担心内存泄露)。

通过hasArray检查一个ByteBuf heap based还是direct buffer。

### 创建ByteBuf

ByteBuf提供了两个工具类来创建ByteBuf，分别是支持池化的Pooled和普通的Unpooled。Pooled缓存了ByteBuf的实例，提高性能并且减少内存碎片。它使用Jemalloc来高效的分配内存。

如果在Channel中我们可以通过channel.alloc()来拿到ByteBufAllocator，具体它使用Pool还是Unpool，Directed还是Heap取决于程序的配置。

### 索引的标记与恢复

markReaderIndex和resetReaderIndex是一个成对的操作。markReaderIndex可以打一个标记，调用resetReaderIndex可以把readerIndex重置到原来打标记的位置。

### 空间释放

discardReadByte可以把读过的空间释放，这时buffer的readerIndex置为0，可写空间和writerIndex也会相应的改变。discardReadBytes在内存紧张的时候使用用，但是调用该方法会伴随buffer的内存整理的。这是一个expensive的操作。

clear是把readerIndex和writerIndex重置到0。但是，它不会进行内存整理，新写入的内容会覆盖掉原有的内容。

### ByteBuf的派生与复制

派生操作會产生一个新的ByteBuf实例。这里的新指得是ByteBuf的引用是新的所有的index也是新的。但是它们共用着一套底层存储。派生函数：

- duplicate()
- slice()
- slice(int, int)
- readSlice(int)
- retainedDuplicate()
- retainedSlice()
- retainedSlice(int, int)
- readRetainedSlice(int)

如果想要复制一个*全新*的ByteBuffer请使用copy，这会完全的复制一个新的ByteBuf出来。

### 引用计数

引用计数记录了当前ByteBuf被引用的次数。新建一个ByteBuf它的refCnt是1，当refCnt == 0时，这个ByteBuf即可被回收。

引用技术主要用于内存泄露的判断，Netty提供了内存泄露检测工具。通过使用参数`-Dio.netty.leakDetectionLevel=${level}`可以配置检测级别：

- 禁用（DISABLED： 完全禁止泄露检测，省点消耗。
- 简单（SIMPLE）: 默认等级，告诉我们取样的1%的ByteBuf是否发生了泄露，但总共一次只打印一次，看不到就没有了。
- 高级（ADVANCED）: 告诉我们取样的1%的ByteBuf发生泄露的地方。每种类型的泄漏（创建的地方与访问路径一致）只打印一次。对性能有影响。
- 偏执（PARANOID）: 跟高级选项类似，但此选项检测所有ByteBuf，而不仅仅是取样的那1%。对性能有绝大的影响。

### 查询

很多时候需要从ByteBuf中查找特定的字符，比如LineBasedFrameDecoder需要在ByteBuf中查找'\r\n'。ByteBuf提供了简单的indexOf这样的函数。同时也可以使用ByteProcesser来查找。

以下[gist](https://gist.github.com/focusj/30c39bf4359df260bd7b629c2b8b7198)提供了一些example。
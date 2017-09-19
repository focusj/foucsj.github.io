# Netty Notes

## 

### ctx.write vs ctx.writeAndFlush vs channel.write

ctx.write: pass through pipeline
flush: flush msg in buffer to channel(flush会有IO操作，所以控制flush的调用次数)
channel.write: 经过所有的pipeline
ctx.writeAndFlush: write + flush

2. 减少GC次数
- voidPromise？？？
- Reactive write。检查Channel isWritable，因为Netty使用了Buffer缓存写操作不停的写可能会导致OOM。
- 配置高低水位线‘

- ByteToMessageDecoder vs ReplayingDecoder
- 优先使用PooledByteBufAllocator
- java -Dio.netty.allocator.numDirectArenas= -Dio.netty.allocator.numHeapArenas
- 堆内存带来额外的内存复制
- 使用ByteProcessor要快于readByte。因为readByte有rangeChecking。（processor 没有range checking，可以被share，容易被JIT inline）
- DefaultByteBufHolder？？？
- zero copy
- Re-use EventLoopGroup if you can
- 如果ChannelHandlers是无状态的，则标记Shareable

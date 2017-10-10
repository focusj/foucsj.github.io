
jemalloc implements three main size class categories as follows (assuming default configuration on a 64-bit system):

Small: [8], [16, 32, 48, ..., 128], [192, 256, 320, ..., 512], [768, 1024, 1280, ..., 3840]
Large: [4 KiB, 8 KiB, 12 KiB, ..., 4072 KiB]
Huge: [4 MiB, 8 MiB, 12 MiB, ...]


jemalloc有2个重点（1、使用arena;2、使用thread cache）




PoolChunk默认的最小内存page是8k，内存map的高度是11层。这么做的好处是可以根据分配的内存快速定位在那分配。

PoolChunk在初始话的时候：chunkSize：16777216，pageSize：8192，maxOrder：11
2^11(mem pages) * 8192(8K) = 16777216(16M)


1) memoryMap[id] = depth_of_id  => it is free / unallocated。(没有被分配过)
2) memoryMap[id] > depth_of_id  => at least one of its child nodes is allocated, so we
cannot allocate it, but some of its children can still be allocated based on their availability。(如果memoryMap[id] > depth_of_id说明这个节点的字节点已经被用过，不能承担承诺的全部内存。但是它的孩子还可用，所以它的深度降了一级。)
3) memoryMap[id] = maxOrder + 1 => the node is fully allocated & thus none of its children can be allocated, it is thus marked as unusable。(已经超过了最大深度，抛错)

内存的回收
Thread cache
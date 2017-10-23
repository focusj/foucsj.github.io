# Netty 内存分配

## Jemalloc 算法

1. 把内存进行分块使用. 按照内存块的大小可以分为Arena, Chunk, Run. 默认情况下会分配两倍CPU个数的Arena.
2. 多线程优化. 当线程申请内存的时候, 按照Round-Robbin算法查找可用的Arena. 这样确保每个Arena上的线程基本是固定的.
3. 线程缓存. 对于小内存的分配优先从线程本地缓存查分配, 如果没有则向新的Chunk申请.

## Netty内存的分配

Arena的结构:
- tinySubPagePool: 用于分配小于512 Byte的内存. 分配额度从16 Byte开始, 每次增长16 Byte.
- smallSubPagePool: 用于分配大于512 Byte的内存. 分配额度从512开始, 每次按倍增长.
- q050: PoolChunkList
- q025: PoolChunkList
- q000: PoolChunkList
- qinit: PoolChunkList
- q075: PoolChunkList
- q100: PoolChunkList

内存的使用分为3个档: 大于 8K, 大于 512Byte, 小于512 Byte. 当大于8K时则从chunk list中查找. 查找顺序为q050 > q025 > q000 > qinit > q075 > q100. 

PoolChunk是以Page组成的, 大小是8K, 每个PoolChunk由2048个Page组成, 并以一个数状结构组织这些节点. 父节点同时管理它字节点的内存. 如果分配8K的内存则从11层开始查找, 并依次类推. 如果某个节点已经被分配, 则它的节点高度加一. 
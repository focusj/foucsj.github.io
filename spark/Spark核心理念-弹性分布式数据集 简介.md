### Resilient Distribute DataSet

RDD是Spark最核心的理念, 要掌握Spark, 首先要把RDD这个概念搞明白. 下面我将尝试去解释一下RDD的概念.



如果你使用过Scala的集合类库, 那么你会发现RDD和它的API非常一致. 在Scala中我们经常使用map, foreach, flatMap等这些函数, 而你和RDD打交道, 无非就是这几个函数. 招式是一样的. 从这个层面看, 你可以把RDD看成是Scala中的一个非变集合(immutable collection).



RDD不同于Scala集合的地方正在于它名字的前两个字母distribute和resilient. 说句玩笑话, 虽然Scala集合和RDD招式上一致(都源于Monad), 但是RDD的内功比Scala集合深厚了不知多少倍. 好了, 我们接着来看刚才提到的RDD最重要的两个特性. Distribute, 这个特性说明RDD可以分布到多台机器上执行; Resilient有可复原, 可恢复的意思, 这表示RDD是可以重新构建的, 具备容错性的.



总结一下, RDD就是一种具有容错性和可并行执行的数据结构. 接下来看一下RDD是如何做到这两点的.



### Distribute

首先, 我们来写一个简单的统计单词数量的示例:

```

val textFile = sc.textFile(inputFile, 3) //读取文件, 指定分区数量

textFile

  .map(_.split(" ")) //把每行按找空格分割, 得到单词的数组

  .map(_.length) //求出每行的单词个数



val wordCount = .reduce(_ + _) //汇总

```

textFile函数是可以携带两个参数的, 第一个是我们要输入的文件, 第二个是partitions的个数. partition这个参数就是和distribute紧密关联的. RDD在执行的时候是以分区为基本单位的, 每个分区持有一定数量的数据, 各个分区在执行的时候是相互独立, 并行执行的. RDD可分区特性为它并行执行提供了前提.



### Resilient

说Resilient之前先稍微铺垫一下. RDD提供了两种API: transformation(转换) & action(执行). 像map, flatMap, filter等这些API都是转换操作. 还有RDD是延迟执行(lazy evaluate)的, 转换操作并不会触发真正的计算, 它只是向RDD提交执行计划. RDD真正的执行是由action函数触发的, action函数有reduce, take, count等. 转换和执行函数还有很多API详细参照[文档](http://spark.apache.org/docs/latest/programming-guide.html).



还有一点, 每个RDD都是只读的. 这是什么意思呢? 就是说, RDD是不可变的, 一经创建不论什么时候读, 在什么地方读, 结果都是一样的. 那问题来了, 既然RDD是只读的, 那它做的那些map, filter到底在干什么呢? 上文也提到了一点*RDD上的transformation是在构建执行计划*; 另外一点是, 建立RDD之间的依赖关系. (这两点是紧密联系的, 这个会在后续分析Spark内部执行流程的文章会提到, 现在大概知道这么回事即可.) 我们可以使用RDD提供的`toDebugString`查看RDD之间的依赖关系. 下边给出了`WordCount`示例所产生的RDD依赖关系:

```

(1) MapPartitionsRDD[5] at map at WordCount.scala:22 []

 |  MapPartitionsRDD[4] at map at WordCount.scala:21 []

 |  MapPartitionsRDD[1] at textFile at WordCount.scala:16 []

 |  /home/focusj/workspace/scala/SparkTour/src/main/scala/lfda/core/WordCount.scala HadoopRDD[0] at textFile at WordCount.scala:16 []

```



那RDD是如何做到容错的呢? 假设一个任务在执行的过程中, 集群中一个节点宕机, 在该机器上运行的任务和数据全部丢失. 这时Spark会立即通知其它节点重新执行该任务. 因为RDD内部记录了足够的信息去恢复这个任务. 这些信息包括RDD之间的依赖关系和执行计划. 所以, RDD是容错的.



到此, 把RDD的基本概念说完了, 下篇会着重解释Spark内部是如何工作的. 从一个RDD Action API的调用开始, 到最终结果输出, Spark都会作哪些工作.


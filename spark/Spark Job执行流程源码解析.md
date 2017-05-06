上一篇文章讲解了RDD的基本概念, 这篇文章尝试分析当Spark拿到一个RDD之后是如何处理它的. 文中会涉及到Spark内部的实现细节, 希望通过本篇文章让大家对Spark有一个更深层次的了解.



## 背景知识

### DAG

DAG(Directed Acyclic Graph) 中文名是有向无环图. DAG是有向无环图(Directed Acyclic Graph)的简称. 在大数据处理领域,

DAG计算模型是指将计算任务在内部分解为若干个子任务, 这些子任务之间由逻辑关系或运行先后顺序等因素被构建成有向无环图. Spark是实现了DAG计算模型的计算框架.



## Spark运行时架构

首先, 先来熟悉一下Spark的运行时架构.



### 驱动器

Spark驱动器就是执行程序main()函数的进程. 驱动器有一下职责:

1. 负责把用户程序转化为多个物理执行单元(task). Task是Spark中最小的工作单元. 具体的步骤是首先它会把用户程序转化成DAG, 然后在把DAG转化成task.

2. 为执行器(Executor)调度任务. 驱动器启动成功后会向Driver程序注册自己, Driver程序保存了所有可用的Executor. 当物理执行计划生成之后它要负责协调哪些任务在哪些Executor执行. 这个过程Driver根据任务基于的数据所在的位置给其分配执行器.



### 执行器

执行器是最"基层", "干活"的进程. Spark应用启动的时候, Executor进程就被同时启动, 知道整个Spark应用关闭, Executor被关闭. 它的具体职责是:

1. 执行Driver交给的任务, 并返回结果.

2. 通过自身的Block Manager为用户程序中要求缓存的RDD提供基于内存的缓存.



### Job执行流程

在Spark中一个Job抽象的执行流程大概就是这样的:Job提交 -> Driver把RDD转化为DAG -> 根据DAG转化为Task -> Task提交给Executor -> Result.

```

+-------+          +----------------+            +-------------+

|  RDD  | --DAG--> |  DAGScheduler  | --Tasks--> |  Executors  |

+-------+          +----------------+            +-------------+

```

简单解释一下涉及到的名词都是什么意思:

Job: 在用户程序中, 每次调用Action函数都会产生一个新的job, 也就是说一个Action都会生成一个job.

Task: Task是Spark中最小的工作单元, Spark中的程序最终都要分解成一个个Task提交到Scheduler.

Stage: Stage对应DAG中的任务单元.



### RDD依赖和DAG的构建

在第一篇文章中, 我提到过执行transformation函数的一个作用是构建RDD之间的依赖关系. 具体来说依赖有宽窄之分, 如果子RDD中的每个分区依赖常数个父RDD中的分区, 我们把这种依赖叫做窄依赖; 如果子RDD中的每个数据分片依赖父RDD的所有分片, 我们把这种依赖叫做宽依赖.



在这儿我们在引入一个新的词汇`lineage`, 在spark中每个RDD都携带自己的lineage. 而lineage就是通过RDD之间的依赖来表示的.



![wide-narrow-dependency](http://upload-images.jianshu.io/upload_images/78847-e790403aedf280bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



我们通过这幅图可以大概看一下宽窄依赖到底是这么回事. 图中矩形框围住的部分是RDD, 实心小矩形是Partition.



接下来我们看一下Spark是如何构建DAG的. 当用户调用Action函数时, 调度器会逆向的遍历该RDD的lineage, 每个stage会尝试尽可能多包含那些连续的窄依赖. 如果当前的Stage向上回溯的过程中遇到了宽依赖, 则当前Stage结束, 一个新的Stage被构建. 第二个Stage是第一个Stage的parent. 还有一种情况也会结束当前Stage, 那就是那个partition已经被计算出来, 换存在内存中, 这种情况下我们就不必作多余的计算了.



## 内部实现

我们依据上边抽象的Job执行流程为依据, 从代码入手看一下Spark内部代码实现.



### 第一个阶段: Job -> Stage.

这个阶段主战场在DAGScheduler, 假设我们调用了reduce函数, reduce内部会调用SparkContext.runJob函数. 在SparkContext内部, 经过一系列函数的调用, 最终通过调用DAGScheduler.runJob函数把Job提交给DAGScheduler.



我们看一下DAGScheduler runjob函数:

```

val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)

waiter.awaitResult() match {

  //...

}

```

首先它先将job提交, 然后创建JobWaiter以阻塞的方式等待job执行结果.

我们接下来看一下DAGScheduler submitJob的过程?



它向DAGSchedulerEventProcessLoop post了一个JobSubmitted事件. DAGSchedulerEventProcessLoop接到JobSubmitted事件之后会调用DAGScheduler的handleJobSubmitted函数. 正是这个函数触发了RDD到DAG的转化. 我们重点来看一下这个函数的实现(删掉了一些我们不关心的代码):

```

private[scheduler] def handleJobSubmitted(...) {

    var finalStage: ResultStage = null

    try {

      finalStage = newResultStage(finalRDD, func, partitions, jobId, callSite)

    } catch {

      case e: Exception =>

        logWarning("Creating new stage failed due to exception - job: " + jobId, e)

        listener.jobFailed(e)

        return

    }

    submitStage(finalStage)

    submitWaitingStages()

  }

```

通过newResultStage我们拿到了最DAG的最后一个Stage(finalStage), 最有一个Stage都是ResultStage. 如果我们反向的遍历就能够知道整个DAG, 这个稍后我们会具体分析newResultStage的实现. 接着看handleJobSubmitted函数, 在拿到DAG的最后一个Stage后, 通过submitStage把它提交, 不出意外submitStage肯定实在向Executor提交Task. 我们按顺序先看newResultStage是如何生成DAG的.

```

private def getParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {

    val parents = new HashSet[Stage]

    val visited = new HashSet[RDD[_]]

    val waitingForVisit = new Stack[RDD[_]]

    def visit(r: RDD[_]) {

      if (!visited(r)) {

        visited += r

        for (dep <- r.dependencies) {

          dep match {

            case shufDep: ShuffleDependency[_, _, _] =>

              parents += getShuffleMapStage(shufDep, firstJobId)

            case _ =>

              waitingForVisit.push(dep.rdd)

          }

        }

      }

    }

    waitingForVisit.push(rdd)

    while (waitingForVisit.nonEmpty) {

      visit(waitingForVisit.pop())

    }

    parents.toList

  }

```

上边背景知识部分我们已经大概知道了Spark是如何划分Stage的. 简单的说就是遇到宽依赖, 就生成新的Stage. 宽依赖会触发shuffle. 我们来看上边代码的visit函数: 拿到RDD的所有的dependency, 如果是窄依赖那么继续查找依赖的RDD的parent; 如果是宽依赖, 则调用getShuffleMapStage把生成的Stage加到当前stage的parents中. 该函数执行完毕, 则整个DAG就构建完成.



看完DAG的构建过程, 我们继续沿着submitStage那条线看下去(以下源码做了部分删减).

```

private def submitStage(stage: Stage) {

  ...

    if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {

      val missing = getMissingParentStages(stage).sortBy(_.id)

      logDebug("missing: " + missing)

      if (missing.isEmpty) {

        logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")

        submitMissingTasks(stage, jobId.get)

      } else {

        for (parent <- missing) {

          submitStage(parent)

        }

        waitingStages += stage

      }

    }

}

```

这个函数非常简单, 先把当前stage的parents提交, 完事儿后在提交自己. 我们重点关注一下: submitMissingTasks.

```

/** Called when stage's parents are available and we can now do its task. */

private def submitMissingTasks(stage: Stage, jobId: Int) {

  ...

  val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()

  val tasks: Seq[Task[_]] = try {

    stage match {

      case stage: ShuffleMapStage =>

        partitionsToCompute.map { id =>

          val locs = taskIdToLocations(id)

          val part = stage.rdd.partitions(id)

          new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,

            taskBinary, part, locs, stage.internalAccumulators)

        }



      case stage: ResultStage =>

        val job = stage.activeJob.get

        partitionsToCompute.map { id =>

          val p: Int = stage.partitions(id)

          val part = stage.rdd.partitions(p)

          val locs = taskIdToLocations(id)



          /**

            * !! 一个ResultTask包含了task的定义(这个task要干什么), 以及在那个partition(part)执行该task

            */

          new ResultTask(stage.id, stage.latestInfo.attemptId,

            taskBinary, part, locs, id, stage.internalAccumulators)

        }

    }

  } catch {

    case NonFatal(e) =>

      abortStage(stage, s"Task creation failed: $e\n${e.getStackTraceString}", Some(e))

      runningStages -= stage

      return

  }



  if (tasks.size > 0) {

    taskScheduler.submitTasks(new TaskSet(

      tasks.toArray, stage.id, stage.latestInfo.attemptId, jobId, properties))

    stage.latestInfo.submissionTime = Some(clock.getTimeMillis())

  } else {

    ...

  }

}

```

首先先找出该stage中所有未执行过的partition, 然后把序列化后的task(taskBinary), partition(part)等信息封装成task. ShuffleMapStage转化成ShuffleMapTask, ResultStage转化成ResultStage. ShuffleMapTask的处理过程会比较复杂一下, 因为会涉及到shuffle的过程. 这个我们后续分析Executor如何执行是再详说. 函数最后我们看到新生成的tasks封装成TaskSet提交给TaskScheduler.

到此我们分析了RDD如何转化成DAG, DAG是如何生成Task并提交的. 接下来分析Executor如何处理Task.



### Executor如何处理Tasks

Task提交成功之后, 我们来看一下都有哪些类参与到了Task执行的过程中? 第一个类就是TaskScheduler, 它主要负责Task的调度工作. 第二个类是SchedulerBackend, SchedulerBackend的作用是向TaskScheduler申请任务, 并分配给Executor去执行. SchedulerBackend可以有不同的实现. 支持本地单机运行的是LocalBackend, 支持Mesos集群运行的是MesosSchedulerBackend, etc. 下面我们分析以本地单机运行为例解释任务执行的过程.

文章上一部分分析到DAGScheduler调用TaskScheduler的submitTasks方法提交Task, 那我们接着来看submitTasks的实现:

```

override def submitTasks(taskSet: TaskSet) {

  ...

  backend.reviveOffers()

}

```

submitTasks最后一步调用了LocalBackend的reviveOffers函数, 这个函数是提醒LocalBackend可以开始执行任务了. 我们继续看reviveOffers的实现:

```

def reviveOffers() {

  val offers = Seq(new WorkerOffer(localExecutorId, localExecutorHostname, freeCores))

  /** 向TaskSchedulerImpl申请Task */

  for (task <- scheduler.resourceOffers(offers).flatten) {

    freeCores -= scheduler.CPUS_PER_TASK

    executor.launchTask(executorBackend, taskId = task.taskId, attemptNumber = task.attemptNumber,

      task.name, task.serializedTask)

  }

}

```

首先, 它先向scheduler申请task. 然后把tasks提交给Executor. 在申请Task的时候, 把自己空闲的cpu个数发送给Scheduler, 以便让Scheduler按资源分配任务. 我们来看Scheduler是如何分配Task的.

```

def resourceOffers(offers: Seq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {

  val shuffledOffers = Random.shuffle(offers)

  val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores))

  val availableCpus = shuffledOffers.map(o => o.cores).toArray

  val sortedTaskSets = rootPool.getSortedTaskSetQueue

  var launchedTask = false



  for (taskSet <- sortedTaskSets; maxLocality <- taskSet.myLocalityLevels) {

    do {

      launchedTask = resourceOfferSingleTaskSet(

          taskSet, maxLocality, shuffledOffers, availableCpus, tasks)

    } while (launchedTask)

  }



  if (tasks.size > 0) {

    hasLaunchedTask = true

  }

  return tasks

}



private def resourceOfferSingleTaskSet(

    taskSet: TaskSetManager,

    maxLocality: TaskLocality,

    shuffledOffers: Seq[WorkerOffer],

    availableCpus: Array[Int],

    tasks: Seq[ArrayBuffer[TaskDescription]]) : Boolean = {

  var launchedTask = false

  for (i <- 0 until shuffledOffers.size) {

    val execId = shuffledOffers(i).executorId

    val host = shuffledOffers(i).host

    if (availableCpus(i) >= CPUS_PER_TASK) {

      try {

        for (task <- taskSet.resourceOffer(execId, host, maxLocality)) {

          tasks(i) += task

          val tid = task.taskId

          taskIdToTaskSetManager(tid) = taskSet

          taskIdToExecutorId(tid) = execId

          executorIdToTaskCount(execId) += 1

          executorsByHost(host) += execId

          availableCpus(i) -= CPUS_PER_TASK

          assert(availableCpus(i) >= 0)

          launchedTask = true

        }

      } catch {

        case e: TaskNotSerializableException =>

          logError(s"Resource offer failed, task set ${taskSet.name} was not serializable")

          // Do not offer resources for this task, but don't throw an error to allow other

          // task sets to be submitted.

          return launchedTask

      }

    }

  }

  return launchedTask

}

```

TaskScheduler每次把所有的TaskSet都取出来, 这些TaskSet都按照一定算法进行了了排序, 排在前边的TaskSet会被优先分配Executor.

在resourceOfferSingleTaskSet函数中, 我们可以知道只有workoffer中Executor的空闲cpu个数大于设定的每个Task需要的cpu数量时, 才把当前的Task添加到tasks列表里.

总得来说任务调度的过程是: Backend向Scheduler发出work offer, worker offer中记录着自己的基本信息, 自己的空闲资源, Scheduler根据worker offer中executor的空闲资源为其分配合适的任务.  



当Backend拿到Task之后, 依次把Task提交给Executor. TaskRunner继承了Runnable, 所以每个Task都是单独的线程去执行.

```

def launchTask(

    context: ExecutorBackend,

    taskId: Long,

    attemptNumber: Int,

    taskName: String,

    serializedTask: ByteBuffer): Unit = {

  val tr = new TaskRunner(context, taskId = taskId, attemptNumber = attemptNumber, taskName,

    serializedTask)

  runningTasks.put(taskId, tr)

  threadPool.execute(tr)

}

```

代码执行到这里, Task开始真正执行.
---
title: Spark-on-Yarn源码解析(四)Spark业务代码的执行及其任务分配调度stage划分
date: 2018年09月04日
tags: [Spark,原理]
categories: Spark-On-Yarn
toc: true
---




# 看看自定义的类

```scala
object WordCount {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("yaoWordCount").setMaster("local[2]")
    val sc = new SparkContext(conf)
    var hadoopRDD: RDD[String] = sc.textFile(args(0))
    var hdfsRDD: RDD[String] = hadoopRDD.flatMap(_.split(""))
    //单词和出现的次数，构建RDD并且调用了他的Transformation
    //返回的是一个hadoopRDD
    //transFormation都是返回的RDD
    var wordAndCount: RDD[(String, Int)] = hdfsRDD.map((_, 1))
    //创建RDD 这里面有两个RDD,一个是hadoopRDD，然后会生成一个paritionRDD
    //savaasTextfile还会产生一个RDD,因为会调用mapPartitons
    //调用RDD的action 开始真正提交任务
    var reducedRDD: RDD[(String, Int)] = wordAndCount.reduceByKey(_ + _)
    reducedRDD.saveAsTextFile(args(1))
    //关闭saprkContext资源
    sc.stop()
  }
}
```
<!--more-->
# sparkContext的初始化

对于Spark程序入口为SparkContext,当我们使用spark-submit/spark-shell等命令来启动一个客户端,客户端与集群需要建立链接，建立的这个链接对象就叫做sparkContext，只有这个对象创建成功才标志这这个客户端与spark集群链接成功。现就将从SparkContext展开来描述一下Spark的任务启动和执行流程。
SparkContext 完成了以下几个主要的功能： 
（1）创建 RDD，通过类似 textFile 等的方法。 
（2）与资源管理器交互，通过 runJob 等方法启动应用。 
（3）创建 DAGScheduler、TaskScheduler 等。 

在SparkContext类中，SparkContext主构造器主要做

我们看一下SparkContext的主构造器

- 调用CreateSparkEnv方法创建SparkEnv(将driver的信息，url，ip等都封装)，SparkEnv中有一个对象ActorSystem
- 创建TaskScheduler ，根据提交任务的URL（如：spark://(.*)"，local[1]等，去创建TaskSchedulerImpl ，然后再创建SparkDeploySchedulerBackend(先后创建driverActor和clientActor)
- 创建DAGScheduler
- TaskScheduler启动，TaskScheduler.start()





## SparkEnv

最终将driver的host,port端口等各种信息都封装到里面

```scala
new SparkEnv(
  executorId,
  actorSystem,
  serializer,
  closureSerializer,
  cacheManager,
  mapOutputTracker,
  shuffleManager,
  broadcastManager,
  blockTransferService,
  blockManager,
  securityManager,
  httpFileServer,
  sparkFilesDir,
  metricsSystem,
  shuffleMemoryManager,
  outputCommitCoordinator,
  conf)
```



## TaskScheduler

在SparkContext类中可以看到，TaskScheduler根据url类型匹配创建TaskSchedulerImpl

```scala
 //TODO 根据提交任务时指定的URL创建相应的TaskScheduler
  private def createTaskScheduler(
      sc: SparkContext,
      master: String): (SchedulerBackend, TaskScheduler) = {
      ...
      case "yarn-standalone" | "yarn-cluster" =>
...
        val scheduler = try {
          val clazz = Class.forName("org.apache.spark.scheduler.cluster.YarnClusterScheduler")
          val cons = clazz.getConstructor(classOf[SparkContext])
          cons.newInstance(sc).asInstanceOf[TaskSchedulerImpl]
          }
      ...
        val backend = try {
          val clazz =
            Class.forName("org.apache.spark.scheduler.cluster.YarnClusterSchedulerBackend")
          val cons = clazz.getConstructor(classOf[TaskSchedulerImpl], classOf[SparkContext])
          cons.newInstance(scheduler, sc).asInstanceOf[CoarseGrainedSchedulerBackend]
        } 
        scheduler.initialize(backend)
        (backend, scheduler)
        ....
        }
```
<!--more-->
可知
TaskScheduler 的实现类`org.apache.spark.scheduler.cluster.YarnScheduler`
TaskSchedulerBacked 的实现类为`org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend`
且TaskScheduler对TaskSchedulerBacked保持了引用
scheduler.initialize(backend)

### 启动TaskScheduler

在Spark的构造函数中,会启动TaskScheduler

```scala
taskScheduler.start()
```

可以看到继承关系

```scala
private[spark] class YarnClusterScheduler(sc: SparkContext) extends YarnScheduler(sc) 
private[spark] class YarnScheduler(sc: SparkContext) extends TaskSchedulerImpl(sc)
```

可以跟踪到，start方法最终调用的是TaskSchedulerImpl里面start方法，在start方法里面

```scala
 override def start() {
    //TODO 首先调用SparkDeploySchedulerBackend的start方法
    backend.start()
    ......
}
```

,这里的backend就是YarnClusterSchedulerBackend，而这个最终继承的是CoarseGrainedSchedulerBackend中start方法

```scala
  override def start() {
  ...
    driverActor = actorSystem.actorOf(
      Props(new DriverActor(properties)), name = CoarseGrainedSchedulerBackend.ACTOR_NAME)
  }
```

获取到spark的配置信息后，会创建driverActor

## DAGScheduler

在SparkContext的构造函数中，会创建DAGScheduler

```scala
    dagScheduler= new DAGScheduler(this)
```

在DAGScheduler构造函数中

```scala
 def this(sc: SparkContext) = this(sc, sc.taskScheduler)
```

可以看到DAGScheduler对TaskScheduler保持了引用

```scala
class DAGScheduler(
    private[scheduler] val sc: SparkContext,
    private[scheduler] val taskScheduler: TaskScheduler,
    listenerBus: LiveListenerBus,
    mapOutputTracker: MapOutputTrackerMaster,
    blockManagerMaster: BlockManagerMaster,
    env: SparkEnv,
    clock: Clock = new SystemClock())
  extends Logging {
  ......
  }
```

- mapOutputTracker：是运行在 Driver 端管理 shuffle 的中间输出位置信息的。 
- blockManagerMaster：也是运行在 Driver 端的，它是管理整个 Job 的 Bolck 信息。

# RDD的构建过程

其中hadoopRDD，hdfsRDD，wordRDD，reduceRDD是经过一系列transformation装换rdd，只有等到action时，才会触发数据的流转

该例的action为saveAsTextFile调用链为

```
   saveAsTextFile()
       saveAsHadoopFile()
            saveAsHadoopFile（重载函数）
                    saveAsHadoopDataset()
                        runJob()之间会调用几个重载函数
                        dagScheduler.runJob()最终调用
```

# 作业提交

# 任务流转

首先注意区分 2 个概述： 
job: 每个 action 都是执行 runJob 方法，可以将之视为一个 job。 
stage：在这个 job 内部，会根据宽依赖，划分成多个 stage。

在action触发后，最最终调用的是DAGScheduler.runJob()

```scala
dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
```

而runJob() 的核心代码为：

```scala
val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
```

即调用 submitJob 方法，我们进一步看看 submitJob()

```scala
  def submitJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): JobWaiter[U] = {
....    
    val jobId = nextJobId.getAndIncrement()
.....

    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
    waiter
  } 
```

submitJob() 方法主要完成了以下 3 个工作： 

- 获取一个新的 jobId 
- 生成一个 JobWaiter，它会监听 Job 的执行状态，而 Job 是由多个 Task 组成的，因此只有当 Job 的所有 Task 均已完成，Job 才会标记成功 
- 最后调用 eventProcessLoop.post() 将 Job 提交到一个队列中，等待处理。这是一个典型的生产者消费者模式。这些消息都是通过 handleJobSubmitted 来处理。

简单看一下 handleJobSubmitted 是如何被调用的。 
首先是 DAGSchedulerEventProcessLoop#onReceive 调用 

```scala
  //TODO 通过模式匹配判断事件的类型 比如任务提交，作业取消 ...
  override def onReceive(event: DAGSchedulerEvent): Unit = event match {
      //TODO 提交计算任务
    case JobSubmitted(jobId, rdd, func, partitions, allowLocal, callSite, listener, properties) =>
      //todo 调用dagScheduler的handlerJobSubmitted方法处理
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, allowLocal, callSite,  listener, properties)
    ... ...   
```

DAGSchedulerEventProcessLoop 是 EventLoop 的子类，它重写了 EventLoop 的 onReceive 方法。以后再分析这个 EventLoop。
onReceive 会调用 handleJobSubmitted。

## stage 的划分

刚才说到 handleJobSubmitted 会从 eventProcessLoop 中取出 Job 来进行处理，处理的第一步就是将 Job 划分成不同的 stage。handleJobSubmitted 主要 2 个工作，一是进行 stage 的划分，这是这部分要介绍的内容；二是创建一个 activeJob，并生成一个任务，这在下一小节介绍。

还是先看看调用链

```
   handleJobSubmitted
       ->newStage()
         ->getParentStages()//此处会遍历RDD所有依赖
            ->getShuffleMapStage()//如果是ShuffleDependency（宽依赖，获取到一个Map）
                ->newOrUsedStage()//这就可以解释我们常说的遇到宽依赖就会划分stage，并且返回stage                
```

所以最终返回的是一个拥有款依赖的           

```scala
  private[scheduler] def handleJobSubmitted(jobId: Int,
      finalRDD: RDD[_],
      func: (TaskContext, Iterator[_]) => _,
      partitions: Array[Int],
      callSite: CallSite,
      listener: JobListener,
      properties: Properties) {
      ...
      //todo 重要：该方法用于划分stage，主要依赖的是finalStage
       finalStage = newStage(finalRDD, partitions.size, None, jobId, callSite)
      .....
    //TODO 集群模式
      activeJobs += job
      ......
    //todo 提交stage
      submitStage(finalStage)
    }
    //TODO  开始向集群提交还在等待的stage
    submitWaitingStages()
  }
```

getParentStages()。 
因为是从最终的 stage 往回推算的，这需要计算最终 stage 所依赖的各个 stage。

```scala
 //TODO 用于获取父stage
  private def getParentStages(rdd: RDD[_], jobId: Int): List[Stage] = {
    val parents = new HashSet[Stage]
    val waitingForVisit = new Stack[RDD[_]]
    def visit(r: RDD[_]) {
      if (!visited(r)) {
        visited + r
        for (dep <- r.dependencies) {
          dep match {
            case shufDep: ShuffleDependency[_, _, _] =>
              //TODO 把宽依赖传进去，获得父stage
              parents += getShuffleMapStage(shufDep, jobId)
            case _ =>
              waitingForVisit.push(dep.rdd)
          }
        }
      }
    }
    waitingForVisit.push(rdd)
    while (!waitingForVisit.isEmpty) {
      visit(waitingForVisit.pop())
    }
    parents.toList
  }
```

## 任务的生成

回到 handleJobSubmitted 中的代码：

```
submitStage(finalStage)
```

submitStage 会提交 finalStage，如果这个 stage 的某些 parentStage 未提交，则递归调用 submitStage()，直至所有的 stage 均已计算完成。

submitStage() 会调用 submitMissingTasks():

submitMissingTasks(stage, jobId.get)

而 submitMissingTasks() 会完成 DAGScheduler 最后的工作：它判断出哪些 Partition 需要计算，为每个 Partition 生成 Task，然后这些 Task 就会封闭到 TaskSet

```scala
 //TODO  DAG提交stage  根据最后一个stage  开始找到第一个stage递归提交stage
  /** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        //TODO 获取他的父stage 没有提交的stage
        val missing = getMissingParentStages(stage).sortBy(_.id)
        //todo 判断父stage是否为空，为空就以为着他是第一stage
        if (missing == Nil) {
       //TODO 开始提交最前面的stage, DAG提交stage给TaskScheduler 会将stage转换成taskSet
          submitMissingTasks(stage, jobId.get)
        } else {
          //TODO 有父stage  就递归提交
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id)
    }
  }

```

submitMissingTasks在最后提交给 TaskScheduler 进行处理

```scala
  //TODO  DAG提交stage给TaskScheduler 会将stage转换成taskSet
  private def submitMissingTasks(stage: Stage, jobId: Int) {
...
//TODO 创建多少个Task

    val tasks: Seq[Task[_]] = if (stage.isShuffleMap) {
      partitionsToCompute.map { id =>
        //TODO 数据存储的最佳位置   移动计算，而不是移动数据
        val locs = getPreferredLocs(stage.rdd, id)
        val part = stage.rdd.partitions(id)
        //TODO 从上游拉取数据
        new ShuffleMapTask(stage.id, taskBinary, part, locs)
      }
    } else {
      val job = stage.resultOfJob.get
      partitionsToCompute.map { id =>
        val p: Int = job.partitions(id)
        val part = stage.rdd.partitions(p)
        val locs = getPreferredLocs(stage.rdd, p)
        //TODO  将数据写入某个介质里面，nosql hdfs 等等
        new ResultTask(stage.id, taskBinary, part, locs, id)
      }
    }

//TODO task的数量最好和分区数一样  如果分区数大于0
 //TODO task的数量最好和分区数一样  如果分区数大于0
    if (tasks.size > 0) {
      logInfo("Submitting " + tasks.size + " missing tasks from " + stage + " (" + stage.rdd + ")")
      stage.pendingTasks ++= tasks

      //TODO 调用taskScheduler的submitTasks提交taskSet 现在将task转换成一个array
taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptId, stage.firstJobId, properties))
      stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
      .....
}
```

## TaskScheduler && TaskSchedulerBackend

上文分析到在 DAGScheduler 中最终会执行 taskScheduler.submitTasks() 方法，我们先简单看一下从这里开始往下的执行逻辑：

```
①taskScheduler.submitTasks()
    ->②schedulableBuilder.addTaskSetManager() 调度模式，是先来先服务还是公平调度模式
    ->③CoarseGrainedSchedulerBackend.reviveOffers() 这个是向driverActor发送消息driverActor ! ReviveOffers
        ->④CoarseGrainedSchedulerBackend.receiveWithLogging 这是driverActor接收消息的部分
            ->⑤CoarseGrainedSchedulerBackend.makeOffers() //case ReviveOffers =>makeOffers()
        这个模式匹配会调用maksOffers方法
                ->⑥launchTasks()调用launchTask向Executor提交task
                    ->⑦ executorData.executorActor ! LaunchTask(new SerializableBuffer(serializedTask))向executor发送序列化好的task，发送一个Task
```

步骤一、二中主要将这组
任务的 TaskSet 加入到一个 TaskSetManager 中。TaskSetManager 会根据数据就近原则为 task 分配计算资源，监控 task 的执行状态等，比如失败重试，推测执行等。 
步骤三、四逻辑较为简单。 
步骤五为每个 task 具体分配资源，它的输入是一个 Executor 的列表，输出是 TaskDescription 的二维数组。TaskDescription 包含了 TaskID, Executor ID 和 task 执行的依赖信息等。 
步骤六、七就是将任务真正的发送到 executor 中执行了，并等待 executor 的状态返回。

​                 


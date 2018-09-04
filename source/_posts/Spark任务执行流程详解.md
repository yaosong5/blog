---
title: Spark-on-Yarn任务执行流程详解
date: 2018年08月06日 22时15分52秒
tags: [Spark,原理]
categories: 大数据
toc: true
---

[TOC]

了解spark-on-yarn,首先我们了解一下yarn提交的流程，俗话说，欲练此功，错了，我们还是先看吧

# yarn任务的提交
YARN 的基本架构和工作流程

![](http://pebgsxjpj.bkt.clouddn.com/15358192541466.jpg)

YARN 的基本架构如上图所示，由三大功能模块组成，分别是 1) RM (ResourceManager) 2) NM (Node Manager) 3) AM(Application Master)
<!--more-->
作业提交
1. 用户通过 Client 向 ResourceManager 提交 Application， ResourceManager 根据用户请求分配合适的 Container, 然后在指定的 NodeManager 上运行 Container 以启动 ApplicationMaster
2. ApplicationMaster 启动完成后，向 ResourceManager 注册自己
3. 对于用户的 Task，ApplicationMaster 需要首先跟 ResourceManager 进行协商以获取运行用户 Task 所需要的 Container，在获取成功后，ApplicationMaster 将任务发送给指定的 NodeManager
4. NodeManager 启动相应的 Container，并运行用户 Task

> [Spark on Yarn](https://www.cnblogs.com/hseagle/p/3728713.html)

在 yarn-cluster 模式下，Spark driver 运行在 application master 进程中，这个进程被集群中的 YARN 所管理，客户端会在初始化应用程序 之后关闭。在 yarn-client 模式下，driver 运行在客户端进程中，application master 仅仅用来向 YARN 请求资源





# ApplicationMaster和Driver的区别

首先区分下 AppMaster 和 Driver，任何一个 yarn 上运行的任务都必须有一个 AppMaster，而任何一个 Spark 任务都会有一个 Driver，Driver 就是运行 SparkContext(它会构建 TaskScheduler 和 DAGScheduler) 的进程，当然在 Driver 上你也可以做很多非 Spark 的事情，这些事情只会在 Driver 上面执行，而由 SparkContext 上牵引出来的代码则会由 DAGScheduler 分析，并形成 Job 和 Stage 交由 TaskScheduler，再由 TaskScheduler 交由各 Executor 分布式执行。

所以 Driver 和 AppMaster 是两个完全不同的东西，Driver 是控制 Spark 计算和任务资源的，而 AppMaster 是控制 yarn app 运行和任务资源的，只不过在 Spark on Yarn 上，这两者就出现了交叉，而在 standalone 模式下，资源则由 Driver 管理。在 Spark on Yarn 上，Driver 会和 AppMaster 通信，资源的申请由 AppMaster 来完成，而任务的调度和执行则由 Driver 完成，Driver 会通过与 AppMaster 通信来让 Executor 的执行具体的任务。






# spark-submit命令

```bash
$SPARK_HOME/bin/spark-submit \
--master yarn \ //提交模式 yarn
--deploy-mode cluster \ //运行的模式，还有一种client模式，但大多用于调试，此处使用cluster模式
--class me.yao.spark.me.yao.spark.WordCount \ //提交的任务
--name "wc" \ //任务名字
--queue root.default \ //提交的队列
--driver-memory 3g \ //为driver申请的内存
--num-executors 1 \ //executors的数量，可以理解为线程数，对应yarn中的Container个数
--executor-memory 6g \ //为每一个executor申请的内存
--executor-cores 4 \ //为每一个executor申请的core
--conf spark.yarn.driver.memoryOverhead=1g \ //driver可使用的非堆内存，这些内存用于如VM，字符 串常量池以及其他额外本地开销等
--conf spark.yarn.executor.memoryOverhead=2g \ //每个executor可使用的非堆内存，这些内存用于如 VM，字符串常量池以及其他额外本地开销等
```
这是通常我们提交spark程序的submit命令，以此为切入点，对spark程序的运行流程做一个跟踪和分析。


## 查看spark-submit脚本
查看spark-submit脚本的信息，初步可以看到submit启动的类为**org.apache.spark.deploy.SparkSubmit**，更多细节其实不重要（开个开玩，极客可以求甚解）如果觉得要深究一下为什么是submit的main方法的可以参考一下[spark on yarn 作业提交源码分析](http://flume.cn/2017/02/28/spark-on-yarn-%E4%BD%9C%E4%B8%9A%E6%8F%90%E4%BA%A4%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
![](http://pebgsxjpj.bkt.clouddn.com/15359432887877.jpg)
接下来查看该类内部的处理逻辑
SparkSumbmit的类（为了简洁和文章篇幅，只保留了关键流程的信息）

```scala
  def main(args: Array[String]): Unit = {
    val appArgs = new SparkSubmitArguments(args)
    if (appArgs.verbose) {
      printStream.println(appArgs)
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
    }
  }
......

  private[spark] def submit(args: SparkSubmitArguments): Unit = {
    val (childArgs, childClasspath, sysProps, childMainClass) = 
prepareSubmitEnvironment(args)
.....
.....
 runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
}

 private[spark] def prepareSubmitEnvironment(args: SparkSubmitArguments)
      : (Seq[String], Seq[String], Map[String, String], String) = {
      ......
      // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    if (isYarnCluster) {
      childMainClass = "org.apache.spark.deploy.yarn.Client"
      .......
      }
//在submit方法中最终调用的是
runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
try {
      mainClass = Class.forName(childMainClass, true, loader)
    } catch {
    ......
    System.exit(CLASS_NOT_FOUND_EXIT_STATUS)
    }
    // SPARK-4170
    private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sysProps: Map[String, String],
      childMainClass: String,
      verbose: Boolean): Unit = {
    ... ...
    
    mainClass = Class.forName(childMainClass, true, loader)
    ... ...
    val mainMethod = mainClass.getMethod("main", new Array[String](0).getClass)
    ... ...
    mainMethod.invoke(null, childArgs.toArray)
    ... ...
    }
```


通过上面的流程可以看到，这样一个调用链(未特殊表明类名，表明为该步上一步的同一类)，我们代码简化一下，看得舒心明了，再配上解说

    submit.main()
        ->submit()模式匹配到该方法，因为我们就是submit提交任务
            ->prepareSubmitEnvironment()该方法中指明了要启动的类，就是大明湖畔的Client
            ->runMain()通过上步指定的类，然后通过反射调用main方法
        
既然我们的线路走到`org.apache.spark.deploy.yarn.Client`        ，那我们再去这个类一看究竟

# org.apache.spark.deploy.yarn.Client
话不多说，先上源码，当然还是简洁版本的
这儿我先上一下最简洁的调用链。

    Client.main()
        ->new Client().run()
             ->monitorApplication(submitApplication())
                ->submitApplication()
                    ->createContainerLaunchContext()会封装一些启动信息如我们启动的类 --class
                        ->userClass
                        ->amArgs
                        ->commands
                        ->printableCommands
                        ->amClass applicationMaster启动的真实类
                        
                    ->createApplicationSubmissionContext()
                        ->Records.newRecord(classOf[Resource])启动
                    ->yarnClientImpl.submitApplication(appContext)
                    
                    
                    
                


最终是调用的client里面main方法->run->
monitorApplication(submitApplication())


```scala
object Client extends Logging {
  def main(argStrings: Array[String]) {
    ... ...
    val sparkConf = new SparkConf
    val args = new ClientArguments(argStrings, sparkConf)
    new Client(args, sparkConf).run()
    ... ...
  }
}

... ...
def run(): Unit = {
    val (yarnApplicationState, finalApplicationStatus) = monitorApplication(submitApplication())
    }
... ...
def submitApplication(): ApplicationId = {
    // TODO: 初始化并且启动client
    yarnClient.init(yarnConf)
    yarnClient.start()
    // TODO: 准备提交请求到resouceManager
    val newApp = yarnClient.createApplication()
    val newAppResponse = newApp.getNewApplicationResponse()
    val appId = newAppResponse.getApplicationId()
    // TODO: 检查集群的内存是否满足当前的任务要求
    verifyClusterResources(newAppResponse)
    // TODO:  设置适当上下文环境来启动applicationMaster
    val containerContext = createContainerLaunchContext(newAppResponse)
    val appContext = createApplicationSubmissionContext(newApp, containerContext)
    // TODO: 提交application
    yarnClient.submitApplication(appContext)
    appId
  }
   private def createContainerLaunchContext(newAppResponse: GetNewApplicationResponse)
    : ContainerLaunchContext = {
    ... ...
          val userClass =
      if (isClusterMode) {
        Seq("--class", YarnSparkHadoopUtil.escapeForShell(args.userClass))
      } else {
        Nil
      }
      ...
       val amClass =
      if (isClusterMode) {
Class.forName("org.apache.spark.deploy.yarn.ApplicationMaster").getName
      } else {
Class.forName("org.apache.spark.deploy.yarn.ExecutorLauncher").getName
      }
          val amArgs =
      Seq(amClass) ++ userClass ++ userJar ++ primaryPyFile ++ pyFiles ++ userArgs ++
        Seq(
          "--executor-memory", args.executorMemory.toString + "m",
          "--executor-cores", args.executorCores.toString,
          "--num-executors ", args.numExecutors.toString)

    val commands = prefixEnv ++ Seq(YarnSparkHadoopUtil.expandEnvironment(Environment.JAVA_HOME) + "/bin/java", "-server"
      ) ++
      javaOpts ++ amArgs ++
    ... ...
     val printableCommands = commands.map(s => if (s == null) "null" else s).toList
    amContainer.setCommands(printableCommands)
    }
    ... ...
def createApplicationSubmissionContext(
      newApp: YarnClientApplication,
      containerContext: ContainerLaunchContext): ApplicationSubmissionContext = {
    val appContext = newApp.getApplicationSubmissionContext
    appContext.setApplicationName(args.appName)
    appContext.setQueue(args.amQueue)
    appContext.setAMContainerSpec(containerContext)
    appContext.setApplicationType("SPARK")
    sparkConf.getOption("spark.yarn.maxAppAttempts").map(_.toInt) match {
      case Some(v) => appContext.setMaxAppAttempts(v)
      case None => logDebug("spark.yarn.maxAppAttempts is not set. " +
          "Cluster's default value will be used.")
    }
   
    val capability = Records.newRecord(classOf[Resource])
    capability.setMemory(args.amMemory + amMemoryOverhead)
    capability.setVirtualCores(args.amCores)
    appContext.setResource(capability)
    appContext
  }

//yarnClient.submitApplication(appContext)提交的真实处  
@Override
  public ApplicationId
      submitApplication(ApplicationSubmissionContext appContext)
          throws YarnException, IOException {
...
//此处通过yarn的协议对applicationMaster进行提交和启动 （此处为个人理解有疑惑，如有错误，还望留言分享，会立即作出更正）
    SubmitApplicationRequest request =
        Records.newRecord(SubmitApplicationRequest.class);
    request.setApplicationSubmissionContext(appContext);
...
```
此处client的事情都已经做完了，请摄影师将镜头切换到applicationMaster




### 小细节用户业务代码信息的封装及流转

我们提交的class的封装流程
    
    ->sublimit的prepareSubmitEnvironment中封装到childArgs中--class
    ->传入到client的构造函数里面作为clientArgs，将其封装到userClass属性里面
    
    在submitApplication中createContainerLaunchContext会将其通过重新封到userClass
        userClass->amArgs->commands->printableCommands
        ->amContainer.setCommands(printableCommands)
        在此，createContainerLaunchContext方法接收到amContainer赋名为containerContext传递给createApplicationSubmissionContext(..,containerContext)
        
    那么在createApplicationSubmissionContext中又有哪些惊天变化（其实并没有）
    
    appContext.setAMContainerSpec(containerContext)
    那么appContext作为createApplicationSubmissionContext方法返回值，由appContext接收，看码
    
    appContext = createApplicationSubmissionContext(newApp, containerContext)
    最后，由yarnClientImpl提交
    yarnClient.submitApplication(appContext)
    码又来了，最终执行的是
    SubmitApplicationRequest request =Records.newRecord(SubmitApplicationRequest.class);
          

# 启动applicationMaster
对于client的封装，对于applicationMaster需要启动的信息(如资源信息)及用户提交的业务代码（wordcount的类信息）信息都已经封装到appContext，并且传递到applicationmaster，那么来看看applicationMaster的执行流程。
程序调用结构

    ApplicationMaster.main()
        ->run()
            ->runDriver()
                ->run()
                    ->startUserApplication()
                        //启动userClass
                        ->userClassLoader.loadClass(args.userClass)
          .getMethod("main", classOf[Array[String]])
                         ->mainMethod.invoke(null, mainArgs)
                    runAMActor()
                    registerAM()
                        ->yarnRmClient.register()->return new YarnAllocater(......)
                        ->yarnAllocator.allocateResources()
                            ->yarnAllocator.handleAllocatedContainers()
                            //启动executor
                            ->yarnAllocator.runAllocatedContainers(containersToUse)
                            

runAllocatedContainers(containersToUse)是去启动 executor，最终真正执行启动Container的是在 ExecutorRunnable.run()中。
创建了 NMClient 客户端调用提供的 API 最终实现在 NM 上启动 Container，**具体如何启动 Container 将在后文中进行介绍。**
    
launcherPool线程池会将container，driver等相关信息封装成ExecutorRunnable对象，通过ExecutorRunnable启动新的container以运行executor。在此过程中，指定启动executor的类是
org.apache.spark.executor.CoarseGrainedExecutorBackend。[spark yarn cluster 模式下任务提交和计算流程分析](https://www.cnblogs.com/superhedantou/p/7688367.html)

## 程序的细节

```scala
def main(args: Array[String]) = {
    SignalLogger.register(log)
    val amArgs = new ApplicationMasterArguments(args)
    SparkHadoopUtil.get.runAsSparkUser { () =>
      master = new ApplicationMaster(amArgs, new YarnRMClient(amArgs))
      System.exit(master.run())
    }
  }
  
  
  ......
  final def run(): Int = {
  ....
    if (isClusterMode) {
        runDriver(securityMgr)
      } else {
        runExecutorLauncher(securityMgr)
      }
...
}

  private def runDriver(securityMgr: SecurityManager): Unit = {
    addAmIpFilter()
    // TODO:  启动我们自定的类，也就是启动submit里面的--class的东西
    userClassThread = startUserApplication()
    val sc = waitForSparkContextInitialized()
...
  actorSystem = sc.env.actorSystem
  runAMActor(
    sc.getConf.get("spark.driver.host"),
    sc.getConf.get("spark.driver.port"),
    isClusterMode = true)
  registerAM(sc.ui.map(_.appUIAddress).getOrElse(""), securityMgr)
  userClassThread.join()
    ...
  }
```


在ApplicationMasterArguments设置了要启动的信息

```scala
class ApplicationMasterArguments(val args: Array[String]) {
  var userJar: String = null
  var userClass: String = null
  var primaryPyFile: String = null
  var pyFiles: String = null
  var userArgs: Seq[String] = Seq[String]()
  var executorMemory = 1024
  var executorCores = 1
  var numExecutors = DEFAULT_NUMBER_EXECUTORS
  ......
  }
```

startUserApplication 主要执行了调用用户的代码，以及创建了一个 spark driver 的进程。 
Start the user class, which contains the spark driver, in a separate Thread.

```scala
private def startUserApplication(): Thread = {
  val classpath = Client.getUserClasspath(sparkConf)
val urls = classpath.map { entry =>
  new URL("file:" + new File(entry.getPath()).getAbsolutePath())
}
val userClassLoader =
...
// TODO:  userClass就是submit里面的--class 提交的类
val mainMethod = userClassLoader.loadClass(args.userClass)
  .getMethod("main", classOf[Array[String]])
userThread.setContextClassLoader(userClassLoader)
userThread.setName("Driver")
userThread.start()
userThread
}
```
从`userThread.setName("Driver")`也可以看出创建的是名为driver的进程

registerAM 向 resourceManager 中正式注册 applicationMaster。注册applicationMaster 以后，并且分配资源，这样，用户代码就可以执行了，任务切分、调度、执行。

然后，用户代码中的 action 会调用 SparkContext 的 runJob，SparkContext 中有很多个 runJob，但最后都是调用 DAGScheduler 的 runJob

```scala
    // registerAM
     private def registerAM(uiAddress: String, securityMgr: SecurityManager) = {
     .....
    allocator = client.register(yarnConf,
      if (sc != null) sc.getConf else sparkConf,
      if (sc != null) sc.preferredNodeLocationData else Map(),
      uiAddress,
      historyAddress,
      securityMgr)
      //为exector分配资源
    allocator.allocateResources()
    reporterThread = launchReporterThread()
    ......
  }
```

## 分配资源部分

# spark业务代码的执行

##  看看自定义的类


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

## sparkContext的初始化

对于Spark程序入口为SparkContext,当我们使用spark-submit/spark-shell等命令来启动一个客户端,客户端与集群需要建立链接，建立的这个链接对象就叫做sparkContext，只有这个对象创建成功才标志这这个客户端与spark集群链接成功。现就将从SparkContext展开来描述一下Spark的任务启动和执行流程。
SparkContext 完成了以下几个主要的功能： 
（1）创建 RDD，通过类似 textFile 等的方法。 
（2）与资源管理器交互，通过 runJob 等方法启动应用。 
（3）创建 DAGScheduler、TaskScheduler 等。 


在SparkContext类中，SparkContext主构造器主要做

我们看一下SparkContext的主构造器

-  调用CreateSparkEnv方法创建SparkEnv(将driver的信息，url，ip等都封装)，SparkEnv中有一个对象ActorSystem
- 创建TaskScheduler ，根据提交任务的URL（如：spark://(.*)"，local[1]等，去创建TaskSchedulerImpl ，然后再创建SparkDeploySchedulerBackend(先后创建driverActor和clientActor)
- 创建DAGScheduler
- TaskScheduler启动，TaskScheduler.start()







### SparkEnv

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



### TaskScheduler

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
可知
TaskScheduler 的实现类`org.apache.spark.scheduler.cluster.YarnScheduler`
TaskSchedulerBacked 的实现类为`org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend`
且TaskScheduler对TaskSchedulerBacked保持了引用
scheduler.initialize(backend)
#### 启动TaskScheduler
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

### DAGScheduler
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
* mapOutputTracker：是运行在 Driver 端管理 shuffle 的中间输出位置信息的。 
* blockManagerMaster：也是运行在 Driver 端的，它是管理整个 Job 的 Bolck 信息。

## RDD的构建过程
其中hadoopRDD，hdfsRDD，wordRDD，reduceRDD是经过一系列transformation装换rdd，只有等到action时，才会触发数据的流转

该例的action为saveAsTextFile调用链为

       saveAsTextFile()
           saveAsHadoopFile()
                saveAsHadoopFile（重载函数）
                        saveAsHadoopDataset()
                            runJob()之间会调用几个重载函数
                            dagScheduler.runJob()最终调用
                            
                            
## 作业提交

### 任务流转
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

* 获取一个新的 jobId 
* 生成一个 JobWaiter，它会监听 Job 的执行状态，而 Job 是由多个 Task 组成的，因此只有当 Job 的所有 Task 均已完成，Job 才会标记成功 
* 最后调用 eventProcessLoop.post() 将 Job 提交到一个队列中，等待处理。这是一个典型的生产者消费者模式。这些消息都是通过 handleJobSubmitted 来处理。

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

### stage 的划分
刚才说到 handleJobSubmitted 会从 eventProcessLoop 中取出 Job 来进行处理，处理的第一步就是将 Job 划分成不同的 stage。handleJobSubmitted 主要 2 个工作，一是进行 stage 的划分，这是这部分要介绍的内容；二是创建一个 activeJob，并生成一个任务，这在下一小节介绍。

还是先看看调用链

       handleJobSubmitted
           ->newStage()
             ->getParentStages()//此处会遍历RDD所有依赖
                ->getShuffleMapStage()//如果是ShuffleDependency（宽依赖，获取到一个Map）
                    ->newOrUsedStage()//这就可以解释我们常说的遇到宽依赖就会划分stage，并且返回stage                
             

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

###任务的生成
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

### TaskScheduler && TaskSchedulerBackend
上文分析到在 DAGScheduler 中最终会执行 taskScheduler.submitTasks() 方法，我们先简单看一下从这里开始往下的执行逻辑：


    ①taskScheduler.submitTasks()
        ->②schedulableBuilder.addTaskSetManager() 调度模式，是先来先服务还是公平调度模式
        ->③CoarseGrainedSchedulerBackend.reviveOffers() 这个是向driverActor发送消息driverActor ! ReviveOffers
            ->④CoarseGrainedSchedulerBackend.receiveWithLogging 这是driverActor接收消息的部分
                ->⑤CoarseGrainedSchedulerBackend.makeOffers() //case ReviveOffers =>makeOffers()
            这个模式匹配会调用maksOffers方法
                    ->⑥launchTasks()调用launchTask向Executor提交task
                        ->⑦ executorData.executorActor ! LaunchTask(new SerializableBuffer(serializedTask))向executor发送序列化好的task，发送一个Task


步骤一、二中主要将这组
任务的 TaskSet 加入到一个 TaskSetManager 中。TaskSetManager 会根据数据就近原则为 task 分配计算资源，监控 task 的执行状态等，比如失败重试，推测执行等。 
步骤三、四逻辑较为简单。 
步骤五为每个 task 具体分配资源，它的输入是一个 Executor 的列表，输出是 TaskDescription 的二维数组。TaskDescription 包含了 TaskID, Executor ID 和 task 执行的依赖信息等。 
步骤六、七就是将任务真正的发送到 executor 中执行了，并等待 executor 的状态返回。

                 



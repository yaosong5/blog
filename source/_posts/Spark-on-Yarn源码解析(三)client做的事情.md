---
title: Spark-on-Yarn源码解析(三)client做的事情
date: 2018年09月04日
tags: [Spark,原理]
categories: Spark-On-Yarn
toc: true
---

[TOC]


spark-on-yarn系列

[Spark-on-Yarn 源码解析 (一)Yarn 任务解析](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析(一)Yarn任务解析/)
[Spark-on-Yarn 源码解析 (二)Spark-Submit 解析](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析(二)Spark-Submit解析/)
[Spark-on-Yarn 源码解析 (三)client 做的事情](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析(三)client做的事情/)
[Spark-on-Yarn 源码解析 (四)Spark 业务代码的执行及其任务分配调度 stage 划分](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析(四)Spark业务代码的执行及其任务分配调度stage划分/)




org.apache.spark.deploy.yarn.Client

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

<!--more-->

最终是调用的client里面main方法->run->

monitorApplication(submitApplication())

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

此处client的事情都已经做完了，请摄影师将镜头切换到applicationMaster



小细节用户业务代码信息的封装及流转

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

启动applicationMaster

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

创建了 NMClient 客户端调用提供的 API 最终实现在 NM 上启动 Container，具体如何启动 Container 将在后文中进行介绍。

    

launcherPool线程池会将container，driver等相关信息封装成ExecutorRunnable对象，通过ExecutorRunnable启动新的container以运行executor。在此过程中，指定启动executor的类是

org.apache.spark.executor.CoarseGrainedExecutorBackend。spark yarn cluster 模式下任务提交和计算流程分析

程序的细节

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

在ApplicationMasterArguments设置了要启动的信息

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

startUserApplication 主要执行了调用用户的代码，以及创建了一个 spark driver 的进程。 

Start the user class, which contains the spark driver, in a separate Thread.

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

从userThread.setName("Driver")也可以看出创建的是名为driver的进程

registerAM 向 resourceManager 中正式注册 applicationMaster。注册applicationMaster 以后，并且分配资源，这样，用户代码就可以执行了，任务切分、调度、执行。

然后，用户代码中的 action 会调用 SparkContext 的 runJob，SparkContext 中有很多个 runJob，但最后都是调用 DAGScheduler 的 runJob

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




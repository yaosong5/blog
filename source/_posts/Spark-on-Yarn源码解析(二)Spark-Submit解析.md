---
title: Spark-on-Yarn源码解析(二)Spark-Submit解析
date: 2018年09月04日
tags: [Spark,原理]
categories: Spark-On-Yarn
toc: true
---

[TOC]

上文我们了解到了yarn的架构和执行任务的流程，接下来我们看看

# spark-submit命令

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

这是通常我们提交spark程序的submit命令，以此为切入点，对spark程序的运行流程做一个跟踪和分析。
<!--more-->
查看spark-submit脚本
![](http://pebgsxjpj.bkt.clouddn.com/15359432887877.jpg)

查看spark-submit脚本的信息，初步可以看到submit启动的类为org.apache.spark.deploy.SparkSubmit，更多细节其实不重要（开个开玩，极客可以求甚解）如果觉得要深究一下为什么是submit的main方法的可以参考一下spark on yarn 作业提交源码分析



接下来查看该类内部的处理逻辑

SparkSumbmit的类（为了简洁和文章篇幅，只保留了关键流程的信息）

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

通过上面的流程可以看到，这样一个调用链(未特殊表明类名，表明为该步上一步的同一类)，我们代码简化一下，看得舒心明了，再配上解说

    submit.main()
        ->submit()模式匹配到该方法，因为我们就是submit提交任务
            ->prepareSubmitEnvironment()该方法中指明了要启动的类，就是大明湖畔的Client
            ->runMain()通过上步指定的类，然后通过反射调用main方法

既然我们的线路走到org.apache.spark.deploy.yarn.Client        ，那我们再去这个类一看究竟，且听下回分解


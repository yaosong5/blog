---
title: Spark-on-Yarn源码解析(一)Yarn任务解析
date: 2018年09月04日
tags: [Spark,原理,Yarn]
categories: Spark-On-Yarn
toc: true
---

[TOC]

spark-on-yarn系列
[Spark-on-Yarn 源码解析①Yarn 任务解析](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析①Yarn任务解析/)
[Spark-on-Yarn 源码解析②Spark-Submit 解析](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析②Spark-Submit解析/)
[Spark-on-Yarn 源码解析③client 做的事情](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析③client做的事情/)
[Spark-on-Yarn 源码解析④Spark 业务代码的执行及其任务分配调度 stage 划分](http://www.gangtieguo.cn/2018/09/04/Spark-on-Yarn源码解析④Spark业务代码的执行及其任务分配调度stage划分/)

了解spark-on-yarn,首先我们了解一下yarn提交的流程，俗话说，欲练此功，错了，我们还是先看吧

# yarn任务的提交
YARN 的基本架构和工作流程

![](http://img.gangtieguo.cn/15358192541466.jpg)

YARN 的基本架构如上图所示，由三大功能模块组成，分别是 1) RM (ResourceManager) 2) NM (Node Manager) 3) AM(Application Master)
<!--more-->
作业提交
1. 用户通过 Client 向 ResourceManager 提交 Application， ResourceManager 根据用户请求分配合适的 Container, 然后在指定的 NodeManager 上运行 Container 以启动 ApplicationMaster
2. ApplicationMaster 启动完成后，向 ResourceManager 注册自己
3. 对于用户的 Task，ApplicationMaster 需要首先跟 ResourceManager 进行协商以获取运行用户 Task 所需要的 Container，在获取成功后，ApplicationMaster 将任务发送给指定的 NodeManager
4. NodeManager 启动相应的 Container，并运行用户 Task


<!--more-->

# Spark-On-Yarn的流程提交

在 **yarn-cluster 模式下，Spark driver 运行在 application master 进程中**，这个进程被集群中的 YARN 所管理，客户端会在初始化应用程序 之后关闭。在 yarn-client 模式下，driver 运行在客户端进程中，application master 仅仅用来向 YARN 请求资源



![](https://img.gangtieguo.cn/006tNbRwgy1fuaxd9man3j31020o60w1.jpg)

Spark Driver首先作为一个ApplicationMaster在YARN集群中启动，客户端提交给ResourceManager的时候，每一个job都会在集群的NodeManager节点上分配一个唯一的ApplicationMaster，由该ApplicationMaster管理全生命周期的应用。具体过程：

1. 由client向ResourceManager提交请求，并上传jar到HDFS上
   这期间包括四个步骤：
   a).连接到RM
   b).从RM的ASM（ApplicationsManager ）中获得metric、queue和resource等信息。
   c). upload app jar and spark-assembly jar
   d).设置运行环境和container上下文（launch-container.sh等脚本)
2. ResouceManager向NodeManager申请资源，创建Spark ApplicationMaster（每个SparkContext都有一个ApplicationMaster）
3. NodeManager启动ApplicationMaster，并向ResourceManager AsM注册
4. ApplicationMaster从HDFS中找到jar文件，启动SparkContext、DAGscheduler和YARN Cluster Scheduler
5. ResourceManager向ResourceManager AsM注册申请container资源
6. ResourceManager通知NodeManager分配Container，这时可以收到来自ASM关于container的报告。（每个container对应一个executor）
7. Spark ApplicationMaster直接和container（executor）进行交互，完成这个分布式任务。



# ApplicationMaster和Driver的区别

首先区分下 AppMaster 和 Driver，任何一个 yarn 上运行的任务都必须有一个 AppMaster，而任何一个 Spark 任务都会有一个 Driver，Driver 就是运行 SparkContext(它会构建 TaskScheduler 和 DAGScheduler) 的进程，当然在 Driver 上你也可以做很多非 Spark 的事情，这些事情只会在 Driver 上面执行，而由 SparkContext 上牵引出来的代码则会由 DAGScheduler 分析，并形成 Job 和 Stage 交由 TaskScheduler，再由 TaskScheduler 交由各 Executor 分布式执行。

所以 Driver 和 AppMaster 是两个完全不同的东西，Driver 是控制 Spark 计算和任务资源的，而 AppMaster 是控制 yarn app 运行和任务资源的，只不过在 Spark on Yarn 上，这两者就出现了交叉，而在 standalone 模式下，资源则由 Driver 管理。在 Spark on Yarn 上，Driver 会和 AppMaster 通信，资源的申请由 AppMaster 来完成，而任务的调度和执行则由 Driver 完成，Driver 会通过与 AppMaster 通信来让 Executor 的执行具体的任务。

> [Spark on Yarn](https://www.cnblogs.com/hseagle/p/3728713.html)



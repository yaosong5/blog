---
title: Spark启动流程及一些小总结
date: 2018年08月06日 22时15分52秒
tags: [Spark,原理]
categories: 大数据
toc: true
---

[TOC]

# spark的架构模型

![](http://img.gangtieguo.cn/006tNbRwly1fuhrnmk1skj310i0ssgma.jpg)

# 角色功能

Driver：以spark-submit提交程序为例，执行该命令的主机为driver（在任意一台安装了spark（spark submit）的机器上启动一个任务的客户端也就是Driver 。客户端与集群需要建立链接，建立的这个链接对象叫做sparkContext，只有这个对象创建成功才标志这这个客户端与spark集群链接成功。SparkContext是driver进程中的一个对象，提交任务的时候，指定了每台机器需要多少个核cores，需要的内存 ）

Master: 给任务提供资源，分配资源，master跟worker通信，报活和更新资源 

Worker:  以子进程的方式启动executor

Executor：Driver提交程序到executor(CoarseGrainedExecutorBankend)，执行任务的进程，exector是task运行的容器

<!-- more -->

## Driver端

driver创建sparkContext于spark集群建立连接，然后向master申请资源，master来调度决定向worker的哪台机器上执行 ，在合适的worker（资源分配策略）上以子进程的方式启动excutor来执行 （Class.forName()会只在自己的进程里面执行，反射是在同一进程 ，启动java子进程的方法 是另一个进程，debug跳不到子进程中去，进程之间进行方法调用只有rpc才可以 ）此过程只是启动executor，最终任务的执行需要等到action触发，提交程序最终也是提交到executor上，在整个过程中，



> driver提交一个application时就会在driver主机生成一个sparkSubmit进程来监控任务的执行情况。action触发程序运行后，executor只会通过driverClinet与driver交互，applicationMaster 不负责具体某个任务的进度 ,只负责资源的调度和监控。driver在action的时候才会把任务提交给executor。

## master与worker交互

worker和master以进程方式启动后 ，worker会向master进行注册,worker在启动的时候，会在spark env中指定worker的资源，如果没指定的话，会默认使用机器的所有核数，所有内存-1G的内存，预留1G给操作系统，然后worker会告知master自己拥有的资源，如核数cores，内存等。在之后的过程中，worker会实时向master发送心跳报活。



## 资源分配策略

spark任务资源分配有两种策略：尽量打散 和尽量集中

以task1需要2core为例：

尽量打散是尽可能的将任务分散到不同的worker来启动exector，机器1分配1core，机器2分配1core

尽量集中是尽可能在更少的机器上来启动worker并启动exector，在分配的机器上尽可能多的分配资源，达到集中在更少的机器上的目的。如在机器1上分配2core（前提是机器1 剩余core数>=2）





# RDD整个运行流程

首先rdd产生过后，经过一系列的处理，构成相互依赖，在有宽依赖的情况下会划分stage，构成一个有向无环图也就是DAG，整个rdd的构建，完成在action触发之前，action触发就会将task提交到executor容器中运行





## Driver与executor交互

在任务提交的时候，会将driver信息封装，先告诉给master，master再告诉给worker，然后worker启动executor进程的时候，会将信息告诉给executor，故而最终exector能获取到driver的信息

在executor创建成功后，就会和driver建立连接，而不再与Master通信，这样是为了减少Master压力，也不让Master成为性能瓶颈。

由于executor有可能在很多台机器上启动，driver无法知晓在哪台机器上启动有exector，所以需要executor和driver进行rpc通信（netty或者akka）主动建立连接。建立通信后exector向driver汇报状态及其执行进度，driver向executor提交计算任务，然后在executor中执行，只有当RDD触发action的时候，driver才将taskset以stage的形式提交任务给executor，任务的最细粒度是task

然后executor就跟driver进行通信，Rdd执行逻辑的时候就不再通过Master，这是为了减少Master压力，也不让Master成为性能瓶颈。



# RDD的构建详细阶段



![](http://img.gangtieguo.cn/006tNbRwly1fuhtcjutpej30e007hmxg.jpg)



## 第一阶段 rdd创建

rdd创建都是在driver中执行，只有运行时有数据流向rdd才会提交到executor。如hdfs中读取数据，会产生很多的RDD….，RDD之间存在着依赖关系， RDD之间的**转换**都是由transformation产生的（故transformation返回对象为RDD），**在action触发后**，DAG就确定了RDD就形成了数据的流向hdfs->RDD1->RDD2->RDD3....->RDDn

## 第二阶段 DAGScheduler

stage的划分，也就是DAGScheduler的执行

stage切分依据：有宽依赖（后面对宽窄依赖有说明）就会切分，更明确的说就是有shuffle的时候切分

宽依赖：遇到shuffle就是宽依赖 
窄依赖：没有shuffle就是窄依赖 

把DAG切分成stage，然后以taskSet(流水线task的集合，所有的task业务逻辑都是一样的，只是计算的数据块不同，因为每一个task只计算一个分区，且taskSet集合中的任务会并行操作）

task由master决定在哪个executor上运行，之后由driver提交task到executor来执行

> 前面的stage先提交，因为stage的划分意味着有shuffle，那么后面stage会依赖前面stage的数据，故此前面stage先提交。



Action触发的时候，DAG就可以确定了，transformation的调用都是在driver中完成的，一旦调用了action，会提交这个任务。

## 第三阶段 TaskScheduler

**TaskScheduler**

**cluster Manager** ：就是Master，启动work上的executor 
**stragling tasks**：例子：当100个任务，99都完成了，剩下了一个任务1，会再启动和剩下的一模一样的任务2，任务1和任务2谁先完成，就用谁的结果



## 第四阶段 任务执行

Block Manager管理数据

也就是action触发，数据开始流通，driver提交task到worker上的executor，执行真正的业务逻辑



计算完成之后，若将这些结果数据collect聚合，则会将所有executor的部分结果聚合在一起，比如count、sum的，多个worker中每一个worker只会保存一部分数据，driver保存了所有数据，、都是在driver里面



# task与读取hdfs数据的关系

task提交之后，如果向hdfs中拉取数据，driver已经将要获取数据的元数据都已经获取到了，task会和分区数挂钩，在executor上一个task读取一个分区。**task读取数据不是一下将所有数据加载到内存里面，是先构建一个迭代器，拿到一条处理一条**。处理完成之后会将数据保存在executor的内存中，如果内存放不下，就将数据持久化到磁盘上


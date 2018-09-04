---
title: Spark-On-Yarn模式
date: 2018年08月06日 22时15分52秒
tags: [Spark,使用]
categories: 大数据
toc: true
---

[TOC]

# SparkOnYarn

# 两种模式区别

## cluster模式：

Driver程序在YARN中运行，应用的运行结果不能在客户端显示，所以最好运行那些将结果最终保存在外部存储介质（如HDFS、Redis、Mysql）而非stdout输出的应用程序，客户端的终端显示的仅是作为YARN的job的简单运行状况。

```bash
./bin/spark-submit --class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
--driver-memory 1g \
--executor-memory 1g \
--executor-cores 2 \
--queue default \
lib/spark-examples*.jar \
10

./bin/spark-submit --class cn.itcast.spark.day1.WordCount \
--master yarn \
--deploy-mode cluster \
--driver-memory 1g \
--executor-memory 1g \
--executor-cores 2 \
--queue default \
/home/bigdata/hello-spark-1.0.jar \
hdfs://master:9000/wc hdfs://master:9000/out-yarn-1

```

<!-- more --> 

## client模式：

Driver运行在Client上，应用程序运行结果会在客户端显示，所有适合运行结果有输出的应用程序（如spark-shell）

```bash
client模式
./bin/spark-submit --class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode client \
--driver-memory 1g \
--executor-memory 1g \
--executor-cores 2 \
--queue default \
lib/spark-examples*.jar \
10

spark-shell必须使用client模式
./bin/spark-shell --master yarn --deploy-mode client

```



# 原理

## cluster模式：

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fuaxd9man3j31020o60w1.jpg)

Spark Driver首先作为一个ApplicationMaster在YARN集群中启动，客户端提交给ResourceManager的每一个job都会在集群的NodeManager节点上分配一个唯一的ApplicationMaster，由该ApplicationMaster管理全生命周期的应用。具体过程：

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

## client模式

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuaxdd9n3tj310c0jbq5y.jpg)

在client模式下，Driver运行在Client上，通过ApplicationMaster向RM获取资源。本地Driver负责与所有的executor container进行交互，并将最后的结果汇总。结束掉终端，相当于kill掉这个spark应用。一般来说，如果运行的结果仅仅返回到terminal上时需要配置这个。

客户端的Driver将应用提交给Yarn后，Yarn会先后启动ApplicationMaster和executor，另外ApplicationMaster和executor都 是装载在container里运行，container默认的内存是1G，ApplicationMaster分配的内存是driver- memory，executor分配的内存是executor-memory。同时，因为Driver在客户端，所以程序的运行结果可以在客户端显 示，Driver以进程名为SparkSubmit的形式存在。
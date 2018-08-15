---
title: Spark任务执行主要流程
date: 2018年08月06日 22时15分52秒
tags: [Spark,原理]
categories: 大数据
toc: true
---

[TOC]



# spark任务执行主要流程 

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuaxw936xxj30e007hmxg.jpg)

<!-- more -->

上图是内核的执行流程 
**DAG**：有向无环图

## **第一阶段：**

从hdfs中读取数据，会产生很多的RDD....,RDD之间存在着依赖关系， 
RDD之间的转换都是由transformation产生的，在action触发后，DAG就确定了（就是所有的RDD），RDD就形成了数据的流向 
hdfs->RDD1->RDD2->RDD3....->RDDn

## **第二阶段**

DAGSchedule 
把DAG切分成stage，然后以taskSet(流水线task的集合，所有的task业务逻辑都是一样的，只是计算的数据不同，因为每一个task只计算一个分区，task由master决定在哪个executor上运行，决定好了，以后提交task还是由driver来执行)方式提交（stage不包括hdfs） 
前面的stage先提交（后面的数据是下游的数据，下游的数据需要依赖上游的数据） 
切分依据：有宽依赖（附件图片有说明）就会切分，更明确的说就是有shuffle的时候切分，一个task就是执行一个分区里面的操作，不同task并行操作，task直接业务逻辑都是样的，只是数据不同 
shuffleRead就是下游向上游拉取数据，类似于reduce的作用，reduce到map的时候，只有等上游数据计算完成了之后才能拉取，所以把上游化为一个stage 
（附件中`切分stage过程`图片有详细说明） 
宽依赖：遇到shuffle就是宽依赖 
窄依赖：没有shuffle就是窄依赖

## **第三个阶段**

TaskScheduler 
**cluster Manager**就是Master，启动work上的executor 
stragling tasks：例子：当100个任务，99都完成了，剩下了一个任务1，会再启动和剩下的一模一样的任务2，任务1和任务2谁先完成，就用谁的结果

## 第四个阶段

Block Manager管理数据

driver再提交task到work上的executor，执行真正的业务逻辑

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuaxwdzmd1j30gf087mxl.jpg)

# 宽依赖，窄依赖

## 窄依赖 narrow dependencies

map,filter 
都是操作的原来分区里面的数据,操作之后也在原来的分区 
三个小分块是RDD的分区，组合起来的大框是RDD 
后面的是子分区，一个父分区只能给一个子，一个子可以对应多个父分区（独生子女，） 
union也是 
join大多数情况下是宽依赖，在一种特殊情况下是窄依赖 
rdd之间的join（详情看附件宽依赖图片），而且只有key value形式的才可以join 
相同key的会join在一起

## 宽依赖 wide dependencies

join，groupBy ，reduceByKey 
groupBy就是按照key来分组的 
多个子女对应独生， 
划分stage的依据就是是否shuffle，是否产生宽依赖 
详细看spark里面的SparkRDD文档 
下图b到g不是一个stage是因为，提前已经分好组，所以是窄依赖，没有stage 
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuaxxmh3m9j30af05rjri.jpg)
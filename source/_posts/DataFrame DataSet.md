---
title:  DataStream DataSet介绍
date: 2018年09月04日 03时02分26秒
tags:  [Spark]
categories: 大数据
toc: true
---

[TOC]

# 什么是DataStream
Discretized Stream是Spark Streaming的基础抽象，代表持续性的数据流和经过各种Spark原语操作后的结果数据流。在内部实现上，DStream是一系列连续的RDD来表示。每个RDD含有一段时间间隔内的数据，如下图：
![](http://pebgsxjpj.bkt.clouddn.com/15360627803806.jpg)


与RDD类似，DataFrame也是一个分布式数据容器。然而DataFrame更像传统数据库的二维表格，除了数据以外，还记录数据的结构信息，即schema。同时，与Hive类似，DataFrame也支持嵌套数据类型（struct、array和map）。从API易用性的角度上 看，DataFrame API提供的是一套高层的关系操作，比函数式的RDD API要更加友好，门槛更低。由于与R和Pandas的DataFrame类似，Spark DataFrame很好地继承了传统单机数据分析的开发体验。


对数据的操作也是按照RDD为单位来进行的
![](http://pebgsxjpj.bkt.clouddn.com/15360627860827.jpg)


计算过程由Spark engine来完成
![](http://pebgsxjpj.bkt.clouddn.com/15360627909583.jpg)


Datasets 与DataFrames 与RDDs的关系
![](http://pebgsxjpj.bkt.clouddn.com/15360698161149.jpg)


Spark引入DataFrame，它可以提供high-level functions让Spark更好的处理结构数据的计算。这让Catalyst optimizer 和Tungsten（钨丝） execution engine自动加速大数据分析。
发布DataFrame之后开发者收到了很多反馈，其中一个主要的是大家反映缺乏编译时类型安全。为了解决这个问题，Spark采用新的Dataset API (DataFrame API的类型扩展)。
Dataset API扩展DataFrame API支持静态类型和运行已经存在的Scala或Java语言的用户自定义函数。对比传统的RDD API，Dataset API提供更好的内存管理，特别是在长任务中有更好的性能提升

# DataStream相关操作

DStream上的原语与RDD的类似，分为Transformations（转换）和OutputOperations（输出）两种，此外转换操作中还有一些比较特殊的原语，如：updateStateByKey()、transform()以及各种Window相关的原语。

## Transformations on DStreams

| Transformation                   | Meaning                                  |
| -------------------------------- | ---------------------------------------- |
| map(func)                        | Return a new DStream by passing each  element of the source DStream through a function func. |
| flatMap(func)                    | Similar to map, but each input item can  be mapped to 0 or more output items. |
| filter(func)                     | Return a new DStream by selecting only  the records of the source DStream on which func returns true. |
| repartition(numPartitions)       | Changes the level of parallelism in this  DStream by creating more or fewer partitions. |
| union(otherStream)               | Return a new DStream that contains the  union of the elements in the source DStream and otherDStream. |
| count()                          | Return a new DStream of single-element  RDDs by counting the number of elements in each RDD of the source DStream. |
| reduce(func)                     | Return a new DStream of single-element  RDDs by aggregating the elements in each RDD of the source DStream using a  function func (which takes two arguments and returns one). The function  should be associative so that it can be computed in parallel. |
| countByValue()                   | When called on a DStream of elements of  type K, return a new DStream of (K, Long) pairs where the value of each key  is its frequency in each RDD of the source DStream. |
| reduceByKey(func, [numTasks])    | When called on a DStream of (K, V) pairs,  return a new DStream of (K, V) pairs where the values for each key are  aggregated using the given reduce function. Note: By default, this uses Spark's  default number of parallel tasks (2 for local mode, and in cluster mode the  number is determined by the config property spark.default.parallelism) to do  the grouping. You can pass an optional numTasks argument to set a different  number of tasks. |
| join(otherStream, [numTasks])    | When called on two DStreams of (K, V) and  (K, W) pairs, return a new DStream of (K, (V, W)) pairs with all pairs of  elements for each key. |
| cogroup(otherStream, [numTasks]) | When called on a DStream of (K, V) and  (K, W) pairs, return a new DStream of (K, Seq[V], Seq[W]) tuples. |
| transform(func)                  | Return a new DStream by applying a  RDD-to-RDD function to every RDD of the source DStream. This can be used to  do arbitrary RDD operations on the DStream. |
| updateStateByKey(func)           | Return a new "state" DStream  where the state for each key is updated by applying the given function on the  previous state of the key and the new values for the key. This can be used to  maintain arbitrary state data for each key. |

 
---
title: SparkStreaming消费Kafka数据
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkStreaming]
categories:
toc: true
---

[TOC]

Streaming消费Kafka有两种方式

#### 1、reciver方式

根据时间来划分批次，缺点：有可能一个时间段会出现数据爆炸，有保存log到hdfs机制，但消耗大（zk来管理偏移量）

#### 2、direct方式 1.3.6后推出

executor和kafka的partition是一一对应的（是rdd的分区和kafka对应，如果一个executor的rdd有多个分区，那么一个executor可以对应多个partition）必须自己来管理偏移量，最好把偏移量写在zk里面

<!-- more -->

# 非直连方式 receive方式

 
spark cluster先与kafka集群建立连接 
在每一个executor中创建一个Reciver 
当executor启动kafka启动后，reciver会一直消费kafka中的数据，一直拉，直连是每隔一个时间段去拉取 
这种方式是zookeeper来管理数据的偏移量 
问题:在时间间隔中，executor接收的数据超过了Executor的内存数，会造成数据的丢失 
为了防止数据丢失，可以做checkpoint，或者是记录日志 
可以多个reciver，一个reciver可以指定多个线程 
**可以读读文章 Kafka Intergration Guide 官网链接**

# 直连Direct Approach (No Receives)


B-*代表broker，每一个broker中有三个分区（假设），但是每个broker里面只有一个分区是活着的 
直连是每隔一个时间段去拉取 
对消费数据的位置保证 
会定期实时查询kafka topic+partition的偏移量，会根据偏移量的范围来处理每一个批次， executor不会接收超出接收范围的数据，而是记录下偏移量，下次接着拉取 
一个kafka的partiton对应rdd的一个partition

问题：一个partition只有一个Executor连接上，不能并行读去数据。 
解决办法，可以repartition，分散到多个partition上去读取。 
怎样让executor读取kafka分区里面的数据的速度快？ 
将boker的分区数创建成和worker的数目一样，也就是executor的数目一样，一个executor消费一个分区，这样数据读取比较快。并行读取数据，也可以控制读取的速度。 
这样需要自己管理偏移量，以前的方式是zk管理偏移量 
最好是将偏移量保存到zk里面，不过是自己控制的，防止将偏移量保存到本地宕机无法恢复。 
**需要读博客 Spark streaming整合kafka** 
官方文档显示，一个数据保证只会消费一次，不会重复消费，更加高效 
简化并行，一对一消费，高效，没有reciver作为消费者
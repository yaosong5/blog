---
title:  Spark本地调试远程集群程序
date: 2018年06月21日 22时15分52秒
tags: [Spark]
categories: 大数据
toc: true
---



[TOC]

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fu562ur7pxj317o0s2qfq.jpg)

由于在生产环境中进行调试spark程序需要进行打包和各种跳板机跳转，最好在本地搭一套集群来进行一些代码基础检查。

<!-- more-->

需要将 本地集群中**hdfs-site.xml core.site.xml **拷贝到本地工程的resource文件夹下，这样会应用这些配置，注意要在提交到生产环境的时候，替换成对应环境的配置文件

```scala
val array = Array(
      ("spark.serializer", "org.apache.spark.serializer.KryoSerializer"),
      ("spark.storage.memoryFraction", "0.3"),
      ("spark.memory.useLegacyMode", "true"),
      ("spark.shuffle.memoryFraction", "0.6"),
      ("spark.shuffle.file.buffer", "128k"),
      ("spark.reducer.maxSizeInFlight", "96m"),
      ("spark.sql.shuffle.partitions", "500"),
      ("spark.default.parallelism", "180"),
      ("spark.dynamicAllocation.enabled", "false")
    )
    val conf = new SparkConf().setAll(array)
      .setJars(Array("your.jar"))
    val sparkSession: SparkSession = SparkSession
      .builder
      .appName(applicationName)
      .enableHiveSupport()
      .master("spark://master:7077")
      .config(conf)
      .getOrCreate()
    val sqlContext = sparkSession.sqlContext
    val sparkContext: SparkContext = sparkSession.sparkContext
```


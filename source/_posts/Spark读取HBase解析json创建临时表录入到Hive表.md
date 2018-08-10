---
title: Spark读取HBase解析json创建临时表录入到Hive表
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkSOL]
categories: 大数据
toc: true
---

[TOC]

介绍：主要是读取通过mysql查到关联关系然后读取HBASE里面存放的Json，通过解析json将json数组对象里的元素拆分成单条json,再将json映射成临时表，查询临时表将数据落入到hive表中

注意：查询HBASE的时候，HBase集群的HMaster，HRegionServer需要是正常运行

主要将内容拆分成几块，spark读取HBase，spark解析json将json数组中每个元素拆成一条（比如json数组有10个元素，需要解析平铺成19个json，那么对应临时表中就是19条记录，对应查询插入到hive也就是19条记录）

spark读取本地HBase

<!-- more -->
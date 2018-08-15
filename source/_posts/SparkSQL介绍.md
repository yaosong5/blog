---
title: SparkSQL介绍
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkSQL]
categories: 大数据
toc: true
---

[TOC]

Hive，它是将Hive SQL转换成MapReduce然后提交到集群上执行，大大简化了编写MapReduce的程序的复杂性，由于MapReduce这种计算模型执行效率比较慢。所有Spark SQL的应运而生，它是将Spark SQL转换成RDD，然后提交到集群执行，执行效率非常快！

<!-- more -->

需要将hive-site.xml拷到spark的配置文件夹

#### hive只认latin1编码

linux下的mysql编码是latin1 
windows下的也设置成latin1.如果要和hive搭配使用的话

#### 进入sparksql和hive连接的命令

```
/home/bigdata/apps/spark/bin/spark-sql --master spark://bigdata1:7077 --driver-class-path /home/bigdata/apps/hive/lib/mysql-connector-java-5.1.31-bin.jar

```

需要注意的是，需要是集群模式，--master 等等，还要指定一个jdbc的连接驱动 
sparksql也会走hive的元数据库

**hive语法**在spark-sql下 
'>create table person(id bigint,name string,age int) row format delimited fields terminated by ',';

在hive下： 
load data inpath "hdfs://bigdata1:9000/person.txt" into table person;


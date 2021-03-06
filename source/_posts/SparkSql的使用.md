---
title: SparkSQL使用
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkSQL]
categories: 大数据
toc: true
---

[TOC]

//1.读取数据，将每一行的数据使用列分隔符分割

val lineRDD = sc.textFile("hdfs://bigdata1:9000/person.txt", 1).map(_.split(" "))
<!-- more -->

//2.定义case class（相当于表的schema）

case class Person(id:Int, name:String, age:Int)

//3.导入隐式转换,在当前版本中可以不用导入

import sqlContext.implicits._

//4.将lineRDD转换成personRDD

val personRDD = lineRDD.map(x => Person(x(0).toInt, x(1), x(2).toInt))

//5.将personRDD转换成DataFrame

val personDF = personRDD.toDF

6.对personDF进行处理

\#(SQL风格语法)

personDF.registerTempTable("t_person")

sqlContext.sql("select * from t_person order by age desc limit 2").show

sqlContext.sql("desc t_person").show

val result = sqlContext.sql("select * from t_person order by age desc")

7.保存结果

result.save("hdfs://bigdata1:9000/sql/res1")

result.save("hdfs://bigdata1:9000/sql/res2", "json")

\#以JSON文件格式覆写HDFS上的JSON文件

import org.apache.spark.sql.SaveMode._

result.save("hdfs://bigdata1:9000/sql/res2", "json" , Overwrite)

8.重新加载以前的处理结果（可选）

sqlContext.load("hdfs://bigdata1:9000/sql/res1")

sqlContext.load("hdfs://bigdata1:9000/sql/res2", "json")


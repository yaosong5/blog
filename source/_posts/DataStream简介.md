---
title: DataStream DataSet简介
date: 2018年09月04日
tags: [Spark]
categories: 大数据
toc: true
---

[TOC]
# 什么是DStream
Discretized Stream是Spark Streaming的基础抽象，代表持续性的数据流和经过各种Spark原语操作后的结果数据流。在内部实现上，DStream是一系列连续的RDD来表示。每个RDD含有一段时间间隔内的数据，如下图：
![](http://pebgsxjpj.bkt.clouddn.com/15360627803806.jpg)

对数据的操作也是按照RDD为单位来进行的
![](http://pebgsxjpj.bkt.clouddn.com/15360627860827.jpg)


计算过程由Spark engine来完成
![](http://pebgsxjpj.bkt.clouddn.com/15360627909583.jpg)


Datasets 与DataFrames 与RDDs的关系
![](http://pebgsxjpj.bkt.clouddn.com/15360698161149.jpg)

Spark引入DataFrame，它可以提供high-level functions让Spark更好的处理结构数据的计算。这让Catalyst optimizer 和Tungsten（钨丝） execution engine自动加速大数据分析。
发布DataFrame之后开发者收到了很多反馈，其中一个主要的是大家反映缺乏编译时类型安全。为了解决这个问题，Spark采用新的Dataset API (DataFrame API的类型扩展)。
Dataset API扩展DataFrame API支持静态类型和运行已经存在的Scala或Java语言的用户自定义函数。对比传统的RDD API，Dataset API提供更好的内存管理，特别是在长任务中有更好的性能提升

# DStream相关操作

DStream上的原语与RDD的类似，分为Transformations（转换）和OutputOperations（输出）两种，此外转换操作中还有一些比较特殊的原语，如：updateStateByKey()、transform()以及各种Window相关的原语。

## Transformations on DStreams

| Transformation               | Meaning                              |
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

 

### 特殊的Transformations


1. UpdateStateByKeyOperation

UpdateStateByKey原语用于记录历史记录，上文中Word Count示例中就用到了该特性。若不用UpdateStateByKey来更新状态，那么每次数据进来后分析完成后，结果输出后将不在保存

2. TransformOperation

Transform原语允许DStream上执行任意的RDD-to-RDD函数。通过该函数可以方便的扩展Spark API。此外，MLlib（机器学习）以及Graphx也是通过本函数来进行结合的。



3. WindowOperations

Window Operations有点类似于Storm中的State，可以设置窗口的大小和滑动窗口的间隔来动态的获取当前Steaming的允许状态
![](http://pebgsxjpj.bkt.clouddn.com/15360679610176.jpg)


## Output Operations on DStreams

Output Operations可以将DStream的数据输出到外部的数据库或文件系统，当某个Output Operations原语被调用时（与RDD的Action相同），streaming程序才会开始真正的计算过程。

| Output Operation                    | Meaning                                  |
| ----------------------------------- | ---------------------------------------- |
| print()                             | Prints the first ten elements of every  batch of data in a DStream on the driver node running the streaming  application. This is useful for development and debugging. |
| saveAsTextFiles(prefix, [suffix])   | Save this DStream's contents as text  files. The file name at each batch interval is generated based on prefix and  suffix: "prefix-TIME_IN_MS[.suffix]". |
| saveAsObjectFiles(prefix, [suffix]) | Save this DStream's contents as  SequenceFiles of serialized Java objects. The file name at each batch  interval is generated based on prefix and suffix:  "prefix-TIME_IN_MS[.suffix]". |
| saveAsHadoopFiles(prefix, [suffix]) | Save this DStream's contents as Hadoop  files. The file name at each batch interval is generated based on prefix and  suffix: "prefix-TIME_IN_MS[.suffix]". |
| foreachRDD(func)                    | The most generic output operator that  applies a function, func, to each RDD generated from the stream. This  function should push the data in each RDD to an external system, such as  saving the RDD to files, or writing it over the network to a database. Note  that the function func is executed in the driver process running the  streaming application, and will usually have RDD actions in it that will  force the computation of the streaming RDDs. |

##  用Spark Streaming实现实时WordCount

架构图：

![](http://pebgsxjpj.bkt.clouddn.com/15360681467801.jpg)


1.安装并启动生成者

首先在一台Linux（ip：192.168.10.101）上用YUM安装nc工具

yum install -y nc

 

启动一个服务端并监听9999端口

nc -lk 9999

 

2.编写Spark Streaming程序

```scala

package me.yao.spark.streaming
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
object NetworkWordCount {
    def main(args: Array[String]) {         //设置日志级别
        LoggerLevel.setStreamingLogLevels()     //创建SparkConf并设置为本地模式运行     //注意local[2]代表开两个线程
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")     //设置DStream批次时间间隔为2秒
    val ssc = new StreamingContext(conf, Seconds(2))     //通过网络读取数据
    val lines = ssc.socketTextStream("192.168.10.101", 9999)     //将读到的数据用空格切成单词
    val words = lines.flatMap(_.split(" "))     //将单词和1组成一个pair
    val pairs = words.map(word => (word, 1))     //按单词进行分组求相同单词出现的次数
    val wordCounts = pairs.reduceByKey(_ + _)     //打印结果到控制台
    wordCounts.print()     //开始计算
    ssc.start()     //等待停止
    ssc.awaitTermination()
        }
    }
```

问题：结[果每次在]()Linux段输入的单词次数都被正确的统计出来，但是结果不能累加！如果需要累加需要使用updateStateByKey(func)来更新状态，下面给出一个例子：



```scala
  package me.yao.spark.streaming

  import org.apache.spark.{HashPartitioner, SparkConf}
  import org.apache.spark.streaming.{StreamingContext, Seconds}

  object NetworkUpdateStateWordCount {
  val updateFunc = (iter: Iterator[(String, Seq[Int], Option[Int])]) => {
          //iter.flatMap(it=>Some(it._2.sum + it._3.getOrElse(0)).map(x=>(it._1,x)))
            iter.flatMap{
             case(x,y,z)=>Some(y.sum + z.getOrElse(0)).map(m=>(x, m))}
           }

  def main(args: Array[String]) {
    LoggerLevel.setStreamingLogLevels*()
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkUpdateStateWordCount")
    val ssc = new StreamingContext(conf, Seconds(5))
    //做checkpoint 写入共享存储中
    ssc.checkpoint("c://aaa")
    **val **lines = ssc.socketTextStream("192.168.10.100", 9999)
    //reduceByKey **结果不累加
    //val result = lines.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_)
    //updateStateByKey结果可以累加但是需要传入一个自定义的累加函数：updateFunc
   val results = lines.flatMap(_.split(" ")).map((_,1)).updateStateByKey(updateFunc, new HashPartitioner(ssc.sparkContext.defaultParallelism), true)
    results.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```
---
title: DataStream DataSet简介
date: 2018年09月04日
tags: [Spark]
categories: 大数据
toc: true
---

[TOC]
# 什么是DStream
Discretized Stream是Spark Streaming的基础抽象，代表持续性的数据流和经过各种Spark原语操作后的结果数据流。在内部实现上，DStream是一系列连续的RDD来表示。每个RDD含有一段时间间隔内的数据，如下图：
![](http://pebgsxjpj.bkt.clouddn.com/15360627803806.jpg)

对数据的操作也是按照RDD为单位来进行的
![](http://pebgsxjpj.bkt.clouddn.com/15360627860827.jpg)


计算过程由Spark engine来完成
![](http://pebgsxjpj.bkt.clouddn.com/15360627909583.jpg)


Datasets 与DataFrames 与RDDs的关系
![](http://pebgsxjpj.bkt.clouddn.com/15360698161149.jpg)

Spark引入DataFrame，它可以提供high-level functions让Spark更好的处理结构数据的计算。这让Catalyst optimizer 和Tungsten（钨丝） execution engine自动加速大数据分析。
发布DataFrame之后开发者收到了很多反馈，其中一个主要的是大家反映缺乏编译时类型安全。为了解决这个问题，Spark采用新的Dataset API (DataFrame API的类型扩展)。
Dataset API扩展DataFrame API支持静态类型和运行已经存在的Scala或Java语言的用户自定义函数。对比传统的RDD API，Dataset API提供更好的内存管理，特别是在长任务中有更好的性能提升

# DStream相关操作

DStream上的原语与RDD的类似，分为Transformations（转换）和OutputOperations（输出）两种，此外转换操作中还有一些比较特殊的原语，如：updateStateByKey()、transform()以及各种Window相关的原语。




##  用Spark Streaming实现实时WordCount

架构图：

![](http://pebgsxjpj.bkt.clouddn.com/15360681467801.jpg)


1.安装并启动生成者

首先在一台Linux（ip：192.168.10.101）上用YUM安装nc工具

yum install -y nc

 

启动一个服务端并监听9999端口

nc -lk 9999

 

2.编写Spark Streaming程序

```scala

package me.yao.spark.streaming
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
object NetworkWordCount {
    def main(args: Array[String]) {         //设置日志级别
        LoggerLevel.setStreamingLogLevels()     //创建SparkConf并设置为本地模式运行     //注意local[2]代表开两个线程
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")     //设置DStream批次时间间隔为2秒
    val ssc = new StreamingContext(conf, Seconds(2))     //通过网络读取数据
    val lines = ssc.socketTextStream("192.168.10.101", 9999)     //将读到的数据用空格切成单词
    val words = lines.flatMap(_.split(" "))     //将单词和1组成一个pair
    val pairs = words.map(word => (word, 1))     //按单词进行分组求相同单词出现的次数
    val wordCounts = pairs.reduceByKey(_ + _)     //打印结果到控制台
    wordCounts.print()     //开始计算
    ssc.start()     //等待停止
    ssc.awaitTermination()
        }
    }
```

问题：结[果每次在]()Linux段输入的单词次数都被正确的统计出来，但是结果不能累加！如果需要累加需要使用updateStateByKey(func)来更新状态，下面给出一个例子：



```scala
  package me.yao.spark.streaming

  import org.apache.spark.{HashPartitioner, SparkConf}
  import org.apache.spark.streaming.{StreamingContext, Seconds}

  object NetworkUpdateStateWordCount {
  val updateFunc = (iter: Iterator[(String, Seq[Int], Option[Int])]) => {
          //iter.flatMap(it=>Some(it._2.sum + it._3.getOrElse(0)).map(x=>(it._1,x)))
            iter.flatMap{
             case(x,y,z)=>Some(y.sum + z.getOrElse(0)).map(m=>(x, m))}
           }

  def main(args: Array[String]) {
    LoggerLevel.setStreamingLogLevels*()
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkUpdateStateWordCount")
    val ssc = new StreamingContext(conf, Seconds(5))
    //做checkpoint 写入共享存储中
    ssc.checkpoint("c://aaa")
    **val **lines = ssc.socketTextStream("192.168.10.100", 9999)
    //reduceByKey **结果不累加
    //val result = lines.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_)
    //updateStateByKey结果可以累加但是需要传入一个自定义的累加函数：updateFunc
   val results = lines.flatMap(_.split(" ")).map((_,1)).updateStateByKey(updateFunc, new HashPartitioner(ssc.sparkContext.defaultParallelism), true)
    results.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```


# Dataset
比RDD执行速度快很多倍，占用的内存更小，是从dataFrame发展而来，包含dataFrame 
dataFrame是处理结构化数据，有表头，有类型，

dataSet从1.6.0开始出现，2.0做了重大改进，对dataFrame进行了整合 
dataFrame在1.4系列出现的，现在很多公司都是用的RDD

在spark的命令行里面： 
将dataFrame转成dataSet 
val ds = df.as[person] 
调用dataSet的方法 

```scala
ds.map 
ds.show 
val ds = sqlContext.read.text("hdfs://bigdata1:9000/wc/).as[String] 
val res5 = ds.flatmap(.split(" ")).map((,1)) 
```
flatmap将文本里面的每一行进行切分， 
rest.reduceByKey();会发现dataSet里面没有这个方法，在dataSet里面应该调用更高级的做法 
ds.flatmap(_.split(" ")).groupBy($""value).count.show 或者collect

在import里面打开idea查看类里面有哪些方法。 
在spark1.6里面sqlContext.read....读取的就是dataFrame，和dataSet还未统一，需要将dataFrame用as转为dataSet





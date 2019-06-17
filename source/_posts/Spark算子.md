---
title: Spark算子案例
date: 2018年08月06日 22时15分52秒
tags: [Spark]
categories: 大数据
toc: true
---

[TOC]

# HelloWord？WorldCount

```scala
sc.textfile("hdfs://master:9000/wc").flatMap(_.split("分隔符")).map((_,1)).reduceByKey(_+_).saveAsTextFile("hdfs://master:9000/wcResult")
```


数据最开始在Driver，计算的时候数据会流入worker
当rdd形成过程中，worker的分区中只是预留了存放数据的位置，只有当action触发的时候，worker的分区中才会存在数据

Spark的运算都是通过算子进行RDD的转换及运算，那我们对算子进行简单熟悉[参考RDD算子实例](http://homepage.cs.latrobe.edu.au/zhe/ZhenHeSparkRDDAPIExamples.html)

<!-- more -->





reduceByKey先进行一下combineer 移动计算

groupByKey不好

reduceByKey会在局部先进行一下求和

groupByKey是会将所有的数据放在一个大集合里面，然后再求和 ，会消耗更多的网络带宽，不符合计算本地化





一下一些RDD是给予rdd1来操作的

```scala
val rdd1 = sc.parallelize(List(1,2,3,4,5,6,7,8,9), 2)
```



## mapPartitions

map 是对 rdd 中的每一个元素进行操作，而 mapPartitions(foreachPartition) 则是对 rdd 中的每个分区的迭代器进行操作。如果在 map 过程中需要频繁创建额外的对象 (例如将 rdd 中的数据通过 jdbc 写入数据库, map 需要为每个元素创建一个链接而 mapPartition 为每个 partition 创建一个链接), 则 mapPartitions 效率比 map 高的多。

SparkSql 或 DataFrame 默认会对程序进行 mapPartition 的优化。



## mapPartitionsWithIndex

mapPartitionWithIndex与mapPartition类似，只是会带上分区的序号

把每个partition中的**分区号和对应的值**拿出来, 源码中方法的形式：

```scala
val func(index,Int,iter:Interator[(Int)]):Interator[String] = {
iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}
```

会转换成函数 
函数的形式

```scala
val func = (index: Int, iter: Iterator[(Int)]) => {
  iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}
rdd1.mapPartitionsWithIndex(func).collect
```

![](http://img.gangtieguo.cn/006tNbRwgy1fucjpo4afaj31ik056whp.jpg)

## aggregate (action)

aggregate是一个action操作

源码定义

```scala
def aggregate[U](zeroValue: U)(seqOp: (U, T) ⇒ U, combOp: (U, U) ⇒ U)(implicit arg0: ClassTag[U]): U
```

eqOp 操作会聚合各分区中的元素，然后 combOp 操作把所有分区的聚合结果再次聚合，两个操作的初始值都是 zeroValue.   seqOp 的操作是遍历分区中的所有元素 (T)，第一个 T 跟 zeroValue 做操作，结果再作为与第二个 T 做操作的 zeroValue，直到遍历完整个分区。combOp 操作是把各分区聚合的结果，再聚合。aggregate 函数返回一个跟 RDD 不同类型的值。因此，需要一个操作 seqOp 来把分区中的元素 T 合并成一个 U，另外一个操作 combOp 把所有 U 聚合。

参考[理解 Spark RDD 中的 aggregate 函数](https://blog.csdn.net/qingyang0320/article/details/51603243)

第一个参数：初始值（在进行操作的时候，会默认带入该值进行） 
第二个参数:   是两个函数[每个函数都是2个参数(第一个函数:先对各个分区进行合并, 第二个函数:对各个分区合并后的结果再进行合并)] 

最后得到返回值



> rdd1为上面的rdd1分区函数的结果

```scala
rdd1.aggregate(0)(_+_, _+_)
```

**0 + (0+1+2+3+4 + 0+5+6+7+8+9)**

![](http://img.gangtieguo.cn/006tNbRwgy1fuck6fl2chj30kg02ut96.jpg)

```scala
rdd1.aggregate(7)(_+_, _+_)
```

**7 + (7+1+2+3+4 + 7+5+6+7+8+9)**

![](http://img.gangtieguo.cn/006tNbRwgy1fuck87fxhij30ju02gt92.jpg)



```scala
rdd1.aggregate(0)(math.max(_, _), _ + _)
```

![](http://img.gangtieguo.cn/006tNbRwgy1fuck95wu88j30qo02o74x.jpg)



```Scala
rdd1.aggregate(5)(math.max(_, _), _ + _)
```

**5和1比, 得5再和234比得5 --> 5和6789比,得9 --> 5 + (5+9)**

![](http://img.gangtieguo.cn/006tNbRwgy1fucouprhcjj30s402yjs5.jpg)





```scala
val rdd2 = sc.parallelize(List("q","w","e","r","t","y","u","i","o","p"),2)
```

可以用更加直接的方式验证操作

```Scala
def func2(index: Int, iter: Iterator[(String)]) : Iterator[String] = {
  iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}
```

```Scala
rdd2.aggregate("")(_ + _, _ + _)
rdd2.aggregate("=")(_ + _, _ + _)
rdd2.aggregate("|")(_ + _, _ + _)
```

![](http://img.gangtieguo.cn/006tNbRwgy1fucpd6axxej30os034wf7.jpg)





```scala
val rdd3 = sc.parallelize(List("qazqqw7","jishhrwe9","sdfwezsddf12","12esdww8"),2)
rdd3.aggregate("")((x,y) => math.max(x.length, y.length).toString, (x,y) => x + y)
```

![](http://img.gangtieguo.cn/006tNbRwgy1fucpvxoentj31gw03276d.jpg)



```scala
val rdd4 = sc.parallelize(List("qazqqw7","jishhrwe9","sdfwezsddf12",""),2)
rdd4.aggregate("")((x,y) => math.min(x.length, y.length).toString, (x,y) => x + y)
```

![](http://img.gangtieguo.cn/006tNbRwgy1fucpwdi27nj31g8032abq.jpg)





## aggregateByKey

对每个分区进行计算

```scala
val pairRDD = sc.parallelize(List( ("a",1), ("a", 12), ("b", 4),("c", 17), ("c", 12), ("b", 2)), 2)
def func2(index: Int, iter: Iterator[(String, Int)]) : Iterator[String] = {
  iter.toList.map(x => "[partID:" +  index + ", val: " + x + "]").iterator
}
pairRDD.mapPartitionsWithIndex(func2).collect

```

## combineByKey

reduceByKey aggregateByKey底层都是依赖的combineByKey，combineByKey比较底层的算子 
和reduceByKey是相同的效果

**combineByKey有三个参数**

第一个参数x: **原封不动取出来**  第二个参数:**是函数, 局部运算**, 第三个:是函数, **对局部运算后的结果再做运算**



```scala
val rdd4 = sc.parallelize(List("a","b","c","d","e","f","g","h","i"),2)
val rdd5 = sc.parallelize(List(1,1,2,2,2,1,2,2,2),2)
val rdd6 = rdd5.zip(rdd4)
val rdd7 = rdd6.combineByKey(List(_),(x:List[String],y:String)=>x:+y,(m:List[String],n:List[String])=>m ++ n)
rdd7.collect
```



## reduceByKey

reduceByKey 用于对每个 key 对应的多个 value 进行 merge 操作，最重要的是它能够在本地先进行 merge 操作，并且 merge 操作可以通过函数自定义。

## groupByKey

groupByKey 也是对每个 key 进行操作，但只生成一个 sequence。不会再进行

需要特别注意 “Note” 中的话，它告诉我们：如果需要对 sequence 进行 aggregation 操作（注意，groupByKey 本身不能自定义操作函数），那么，选择 reduceByKey/aggregateByKey 更好。这是因为 groupByKey 不能自定义函数，我们需要先用 groupByKey 生成 RDD，然后才能对此 RDD 通过 map 进行自定义函数操作。



## checkpoint

将rdd内容持久化

```scala
sc.setCheckpointDir("hdfs://master:9000/ck")
val rdd = sc.textFile("hdfs://master:9000/wc").flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_)
rdd.checkpoint
rdd.isCheckpointed
rdd.count
rdd.isCheckpointed
rdd.getCheckpointFile

```

## coalesce, repartition

有时候需要重新设置 Rdd 的分区数量，比如 Rdd 的分区中，Rdd 分区比较多，但是每个 Rdd 的数据量比较小，需要设置一个比较合理的分区。或者需要把 Rdd 的分区数量调大。还有就是通过设置一个 Rdd 的分区来达到设置生成的文件的数量。

如果分区的数量发生激烈的变化，如设置 numPartitions = 1，这可能会造成运行计算的节点比你想象的要少，为了避免这个情况，可以设置 shuffle=true，

那么这会增加 shuffle 操作。

关于这个分区的激烈的变化情况，比如分区数量从父 Rdd 的几千个分区设置成几个，有可能会遇到这么一个错误。

```
java.io.IOException: Unable to acquire 16777216 bytes of memory
```

这个错误只要把 shuffle 设置成 true 即可解决。

当把父 Rdd 的分区数量增大时，比如 Rdd 的分区是 100，设置成 1000，如果 shuffle 为 false，并不会起作用。

这时候就需要设置 shuffle 为 true 了，那么 Rdd 将在 shuffle 之后返回一个 1000 个分区的 Rdd，数据分区方式默认是采用 hash partitioner。

最后来看看 repartition() 方法的源码：





coalesce() 方法的作用是返回指定一个新的指定分区的 Rdd。



```scala
val rdd1 = sc.parallelize(1 to 10, 10)
val rdd2 = rdd1.coalesce(2, false)
rdd2.partitions.length

```

[](https://www.cnblogs.com/fillPv/p/5392186.html)

## collectAsMap 

将其他集合保存为map结构

```scala
val rdd = sc.parallelize(List(("a", 1), ("b", 2)))
rdd.collectAsMap
得到结果
Map(b -> 2, a -> 1)
```



## countByKey

```scala
val rdd1 = sc.parallelize(List(("a", 1), ("b", 2), ("b", 2), ("c", 2), ("c", 1)))
rdd1.countByKey 
统计Key出现的次数
结果 Map(b -> 2, a -> 1, c -> 2) 
```



## countByValue

```scala
val rdd1 = sc.parallelize(List(("a", 1), ("b", 2), ("b", 2), ("c", 2), ("c", 1)))
rdd1.countByValue
结果 (将整个元组作为key)
Map((b,2) -> 2, (c,2) -> 1, (a,1) -> 1, (c,1) -> 1)
```





## filterByRange

```scala
val rdd1 = sc.parallelize(List(("a", 5), ("b", 3), ("c", 4), ("d", 2), ("e", 1)))
val rdd2 = rdd1.filterByRange("b", "d")
rdd2.collect

Array[(String, Int)] = Array((b,3), (c,4), (d,2))

```

## flatMapValues 

压平

```scala
val rdd3 = sc.parallelize(List(("a", "1 2"), ("b", "3 4")))
val rdd4 = rdd3.flatMapValues(_.split(" "))
rdd4.collect

Array[(String, String)] = Array((a,1), (a,2), (b,3), (b,4))

```

## foldByKey

```scala
val rdd1 = sc.parallelize(List("a22", "b232", "c", "d"), 2)
val rdd2 = rdd1.map(x => (x.length, x))
rdd2.collect
结果： Array[(Int, String)] = Array((3,a22), (4,b232), (1,c), (1,d))

val rdd3 = rdd2.foldByKey("")(_+_)
rdd3.collect




结果：将相同key的元组合并在一起，
Array[(Int, String)] = Array((4,b232), (1,cd), (3,a22))

```

## foreach

foreach是针对于每一个元素， 
foreachPartition是针对每一个分区， 
foreachPartition是写入数据库时，可以将在foreachPartition时获得一个数据库连接，通过map方法来将每个分区的全部元素写入到数据库

## foreachPartition

3个分区

```scala
val rdd1 = sc.parallelize(List(1, 2, 3, 4, 5, 6, 7, 8, 9), 3)
rdd1.foreachPartition(x => println(x.reduce(_ + _)))

```

## keyBy

以传入的参数做key

```scala
val rdd1 = sc.parallelize(List("dog", "salmon", "salmon", "rat", "elephant"), 3)
val rdd2 = rdd1.keyBy(_.length)
rdd2.collect
结果 Array((3,dog), (6,salmon), (6,salmon), (3,rat), (8,elephant))

```

## keys values

```scala
val rdd1 = sc.parallelize(List("dog", "tiger", "lion", "cat", "panther", "eagle"), 2)
val rdd2 = rdd1.map(x => (x.length, x))
rdd2.keys.collect
rdd2.values.collect
```






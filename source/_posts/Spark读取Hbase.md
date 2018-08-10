---
title: Spark读取HBase
date: 2018年08月06日 22时15分52秒
tags: [Spark,HBase]
categories: 大数据
toc: true
---

[TOC]

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fu552ugvkxj30ys0han0l.jpg)

Spark读取Hbase

<!-- more -->

# spark配置

首先spark的配置

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

# Hbase配置

```scala
val hBaseConf = HBaseConfiguration.create()

var scan = new Scan();
scan.addFamily(Bytes.toBytes("cf"));
var proto = ProtobufUtil.toScan(scan)
var scanToString = Base64.encodeBytes(proto.toByteArray())
//以为全局扫描的方式
hBaseConf.set(TableInputFormat.SCAN,scanToString)
//如需要设置起止行的话
//scan.setStartRow(Bytes.toBytes("1111111111111"))
//scan.setStopRow(Bytes.toBytes("999999999999999"))
hBaseConf.set("hbase.zookeeper.quorum","zk1,zk2,zk3")
hBaseConf.set("phoenix.query.timeoutMs","1800000")
hBaseConf.set("hbase.regionserver.lease.period","1200000")
hBaseConf.set("hbase.rpc.timeout","1200000")
hBaseConf.set("hbase.client.scanner.caching","1000")
hBaseConf.set("hbase.client.scanner.timeout.period","1200000")
//表名配置
hBaseConf.set(TableInputFormat.INPUT_TABLE,"beehive:a_up_rawdata")
// 从数据源获取数据
val hbaseRDD = sparkContext.newAPIHadoopRDD(hBaseConf,classOf[TableInputFormat],classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],classOf[org.apache.hadoop.hbase.client.Result])
//即可得到读取Hbase查询的RDD
 val hbaseJsonRdd: RDD[String] = hbaseRDD.filter(t =>
        broadCast.value.contains(Bytes.toString(t._2.getRow))
      //********************************操作每个分区的数据********************************
    ).mapPartitions( it=>{
      it.map(x=>x._2).map(hbaseValue => {
        var listBuffer = new ListBuffer[String]()
        //对应的值
        val rowkey = Bytes.toString(hbaseValue.getRow)
        val value: String = Bytes.toString(hbaseValue.getValue(Bytes.toBytes("cf"), Bytes.toBytes("填写获取哪一列")))
        if (null != value ) {
          //如果value不为空则再进行操作
        }
        listBuffer

      })
    }).flatMap(r => r)

//注意map操作是需要函数内部有返回值的，如果只是打印的话，换成foreach算子
    println(s"hbaseJsonRdd.size为：${hbaseJsonRdd.count()}")
    sparkContext.stop()
    sparkSession.close()
    println("ALL 已经关闭，程序终止")
```






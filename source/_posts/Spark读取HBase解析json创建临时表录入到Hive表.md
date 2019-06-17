---
title: Spark读取HBase解析json创建临时表录入到Hive表
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkSQL]
categories: 大数据
toc: true
---

[TOC]

![](http://img.gangtieguo.cn/0069RVTdgy1fu81r6j5e5j30ji09c3z1.jpg)

介绍：主要是读取通过mysql查到关联关系然后读取HBASE里面存放的Json，通过解析json将json数组对象里的元素拆分成单条json,再将json映射成临时表，查询临时表将数据落入到hive表中

注意：查询HBASE的时候，HBase集群的HMaster，HRegionServer需要是正常运行

主要将内容拆分成几块，spark读取HBase，spark解析json将json数组中每个元素拆成一条（比如json数组有10个元素，需要解析平铺成19个json，那么对应临时表中就是19条记录，对应查询插入到hive也就是19条记录）

spark读取本地HBase

<!-- more -->

参考 [Spark读取HBase](http://www.gangtieguo.cn/2018/08/11/Spark读取Hbase/)

# json样例

![](http://img.gangtieguo.cn/006tNbRwgy1fu53tng462j31721e6afd.jpg)



# 读取hbase

hbase里面存放的是身份id作为rowkey来存放的数据

> JSON、JSONObject类包是引用的com.alibaba.fastjson包下的

```scala
 val hbaseJsonRdd: RDD[String] = hbaseRDD.mapPartitions( it=>{
      it.map(x=>x._2).map(hbaseValue => {
        var listBuffer = new ListBuffer[String]()
        //对应的值
        //获取key,也就是身份证号，通过身份证号在广播map中的值 也就是risk_request_id
        val idNum = Bytes.toString(hbaseValue.getRow)
        val json: String = Bytes.toString(hbaseValue.getValue(Bytes.toBytes("cf"), Bytes.toBytes(s"273468436_data")))
        if (null != json ) {
          //********************************获取到json之后进行解析********************************
          try {
            val jSONObject: JSONObject = JSON.parseObject(json)
            if (jSONObject != null) {
                val contactRegion = repostData.getJSONArray("contact_region")
                if (contactRegion != null) {
                  contactRegion.toArray().foreach(v => {
                    val arrays = JSON.parseObject(v.toString)
                    //val map = JSON.toJavaObject(arrays,classOf[util.Map[String,String]])
                    val map: mutable.Map[String, String] = JsonUtils.jsonObj2Map(arrays)
                    //将json 转为Map
                    //将******************************** 日期和requestId request_id封装到 map里面********************************，再将map转为json
                    map.put("region_id", "2")
                    map.put("request_id", "1")
                    map.put("region_create_at", "0000")
                    map.put("region_update_at", "0000")
                    listBuffer += (JsonUtils.map2Json(map))
                  })
                }
              }
            }
          }catch {
            case e: Exception => e.printStackTrace()
          }
        }
        listBuffer
      })
    }).flatMap(r => r)
```

代码中的 jsonObj2Map,map2Json 方法参照 [Json与Scala类型的相互转换处理](http://www.gangtieguo.cn/2018/08/11/Json与Scala类型的一些互相转换处理/)

这里拆分json数组每一个元素为一个json，存放在ListBuffer里面，通过flatMap压平rdd里面的内容。

# 映射临时表

最后将得到的json通过sparkSql创建成临时表

```scala
 val dataFrame: DataFrame = sqlContext.read.json(hbaseJsonRdd)
dataFrame.createOrReplaceTempView("tmp_hbase")
//// 测试

println("++++++++++++++++++++++++++++++hbaseJsonRdd.....创建临时表 测试查询数据  ......++++++++++++++++++++++++++++++")
val df = sqlContext.sql("select * from tmp_hbase limit 1")
df.show(1)
```

# 插入Hive

```scala
sqlContext.sql("insert into ods.ods_r_juxinli_region_n partition(dt='20180101') select region_id as juxinli_region_id,request_id as juxinli_request_id," +
        "region_loc as juxinli_rejion_loc ,region_uniq_num_cnt as juxinli_region_uniq_num_cnt ," +
        "region_call_out_time as juxinli_region_call_out_time,region_call_in_time as juxinli_region_call_in_time,region_call_out_cnt as juxinli_region_call_out_cnt," +
        "region_call_in_cnt as juxinli_region_call_in_cnt,region_avg_call_in_time as juxinli_region_avg_call_in_time,region_avg_call_out_time as juxinli_region_avg_call_out_time," +
        "region_call_in_time_pct as juxinli_region_call_in_time_pct,region_call_out_time_pct as juxinli_region_call_out_time_pct ,region_call_in_cnt_pct as juxinli_region_call_in_cnt_pct," +
        "region_call_out_cnt_pct as juxinli_region_call_out_cnt_pct,region_create_at as juxinli_region_create_at,region_update_at as juxinli_region_update_at from tmp_hbase")
    }
```

**关闭资源**

```scala
sparkContext.stop()
sparkSession.close()
```
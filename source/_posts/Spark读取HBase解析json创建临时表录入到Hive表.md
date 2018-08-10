---
title: Spark读取HBase解析json创建临时表录入到Hive表
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkSOL]
categories: 大数据
toc: true
---

[TOC]

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fu552ugvkxj30ys0han0l.jpg)

介绍：主要是读取通过mysql查到关联关系然后读取HBASE里面存放的Json，通过解析json将json数组对象里的元素拆分成单条json,再将json映射成临时表，查询临时表将数据落入到hive表中

注意：查询HBASE的时候，HBase集群的HMaster，HRegionServer需要是正常运行

主要将内容拆分成几块，spark读取HBase，spark解析json将json数组中每个元素拆成一条（比如json数组有10个元素，需要解析平铺成19个json，那么对应临时表中就是19条记录，对应查询插入到hive也就是19条记录）

spark读取本地HBase

<!-- more -->

参考 [Spark读取HBase](http://www.gangtieguo.cn/2018/08/11/Spark读取Hbase/)

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fu53tng462j31721e6afd.jpg)



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
---
title: Json与Scala类型的相互转换处理
date: 2018年08月06日 22时15分52秒
tags: [Json,Scala]
categories: 大数据
toc: true
---




[TOC]

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fu5817otq1j31kw0mojzv.jpg)

在开发过程中时常会有对json数据的一些处理，现做一些记录

<!-- more -->

```scala
import com.alibaba.fastjson.{JSON, JSONArray, JSONObject}
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.scala.DefaultScalaModule
import net.minidev.json.parser.JSONParser
import scala.collection.JavaConversions.mapAsScalaMap
import scala.collection.mutable
import java.util

/**
  * json utils
  */
object JsonUtils {


  val mapper: ObjectMapper = new ObjectMapper()

  def toJsonString(T: Object): String = {
    mapper.registerModule(DefaultScalaModule)
    mapper.writeValueAsString(T)
  }

  def getArrayFromJson(jsonStr: String) = {
    JSON.parseArray(jsonStr)
  }
  def getObjectFromJson(jsonStr: String): JSONObject = {
    JSON.parseObject(jsonStr)
  }
  /**
    * 配合getObjectFromJson 使用把 JSONObject 变为 map
    * @param jsonObj
    * @return
    */
  def jsonObj2Map(jsonObj:JSONObject): mutable.Map[String, String] = {
    var map = mutable.Map[String, String]()
    val itr: util.Iterator[String] = jsonObj.keySet().iterator()
    while (itr.hasNext) {
      val key = itr.next()
      map += ((key, jsonObj.getString(key)))
    }
    map
  }
  /**
    * json 字符串转成 Map
    * #############有些情况下转换会有问题###############
    * @param json
    * @return
    */
  def json2Map(json: String): mutable.HashMap[String,String] ={
    val map : mutable.HashMap[String,String]= mutable.HashMap()
    val jsonParser =new JSONParser()
    //将string转化为jsonObject
    val jsonObj: JSONObject = jsonParser.parse(json).asInstanceOf[JSONObject]

    //获取所有键
    val jsonKey = jsonObj.keySet()

    val iter = jsonKey.iterator()

    while (iter.hasNext){
      val field = iter.next()
      val value = jsonObj.get(field).toString

      if(value.startsWith("{")&&value.endsWith("}")){
        val value = mapAsScalaMap(jsonObj.get(field).asInstanceOf[util.HashMap[String, String]])
        map.put(field,value.toString())
      }else{
        map.put(field,value)
      }
    }
    map
  }
  /**
    * map 转换成 json 字符串
    * @param map
    * @return
    */
  def map2Json(map : mutable.Map[String,String]): String = {
    import net.minidev.json.{JSONObject}
    import scala.collection.JavaConversions.mutableMapAsJavaMap
    val jsonString = JSONObject.toJSONString(map)
    jsonString
  }
}
```

# 测试实例



```scala
def main(args: Array[String]) {

      val json = "[{\"batchid\":305322456,\"amount\":20.0,\"count\":20},{\"batchid\":305322488,\"amount\":\"10.0\",\"count\":\"10\"}]"
      val array: JSONArray = JsonUtils.getArrayFromJson(json)
      println(array)
      array.toArray().foreach(json=>{
        println(json)
        val jobj = json.asInstanceOf[JSONObject]
        println(jobj.get("batchid"))
      })

      val jsonStr = "{\"batchid\":119,\"amount\":200.0,\"count\":200}"
      val jsonObj: JSONObject = JsonUtils.getObjectFromJson(jsonStr)
      println(jsonObj)

      val jsonObj2: JSONObject = JsonUtils.getObjectFromJson("{'name':'Wang','age':18,'tag1':[{'tn1':'100','tn2':'101','ts':'ts01'},{'tn1':'100','tn2':'101','ts':'ts02'},{'tn1':'100','tn2':'101','ts':'ts03'}]}")
      println(jsonObj2)
}
```


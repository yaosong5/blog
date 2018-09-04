---
title: Hive自定义函数UDF相关
date: 2018年08月06日 22时15分52秒
tags: [Hive]
categories: 大数据
toc: true
---

[TOC]

# UDF开发及使用

1. 打成jar包上传到服务器，将jar包添加到hive的classpath

```sql
或者 hive>add JAR /home/hadoop/udf.jar;
```

1. 创建临时函数与开发好的java class关联

```
Hive>create temporary function runTime as 'me.yao.bigdata.udf.RunTime';
```

即可在hql中使用自定义的函数time() 

Select time(name),age from t_test;

> add jar只在一次会话中生效

<!-- more -->



# transform案例:可以不需要上传jar包

## 1、加载数据

先加载rating.json([链接](https://pan.baidu.com/s/1kJnjv-en3R3HoFB8vhM9AA))文件到hive的一个原始表 rat_json

```sql
create table rat_json(line string) row format delimited;
load data local inpath '/home/bigdata/apps/hive/hivedata/rating.json' into table rat_json;
```

## 2、解析字段

需要解析json数据成四个字段，插入一张新的表 t_rating

```sql
create table t_rating as
select get_json_object(line,'$.movie') as movieid,get_json_object(line,'$.rate')as rate,get_json_object(line,'$.timeStamp')as timestring,get_json_object(line,'$.uid')as uid from rat_json;


```

或者

```sql
insert overwrite table t_rating
select get_json_object(line,'$.movie') as movieid,get_json_object(line,'$.rate')as rate,get_json_object(line,'$.timeStamp')as timestring,get_json_object(line,'$.uid')as uid from rat_json;
```

## 3、转换weekday

使用transform+python的方式去转换unixtime为weekday

先编辑一个python脚本文件

**cd /usr/hive/hivedata/**

**vim weekday_mapper.py**

```python
#!/bin/python

import sys

import datetime

for line in sys.stdin:

  line = line.strip()

  movieid, rating, unixtime,userid = line.split('\t')

  weekday = datetime.datetime.fromtimestamp(float(unixtime)).isoweekday()

  print '\t'.join([movieid, rating, str(weekday),userid])
```

## 将文件加入classpath

保存文件

然后，将文件加入hive的classpath：

```sql
hive>add FILE /usr/hive/hivedata/weekday_mapper.py;
```

## 再创建表

```sql
create TABLE u_data_new as
SELECT
  TRANSFORM (movieid, rate, timestring,uid)
  USING 'python weekday_mapper.py'
  AS (movieid, rate, weekday,uid)
FROM t_rating;
```

## 查询表

```sql
select distinct(weekday) from u_data_new limit 10;
```


---
title: Hive分桶表,分区表简单分析
date: 2018年08月06日 22时15分52秒
tags: [Hive]
categories: 大数据
toc: true
---

[TOC]

![](https://ws2.sinaimg.cn/large/0069RVTdgy1fu81tzpekzj305z06j3yc.jpg)

对于每一个表或者是分区，Hive 可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive 是针对某一列进行分桶。Hive 采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶中。分桶的好处是可以获得更高的查询处理效率。使取样更高效。

分桶依赖于yarn的所以分桶的时候需要启动yarn

<!-- more -->

# 分桶表创建

\#设置变量,设置分桶为true, 设置reduce数量是分桶的数量个数

```sql
set hive.enforce.bucketing = true;
set mapreduce.job.reduces=4;
```

创建表

```sql
create table person_buck(id int,name string,sex string,age int)
clustered by(id) 
sorted by(id DESC)
into 4 buckets
row format delimited
fields terminated by ',';
```



```
开会往创建的分桶表插入数据(插入数据需要是已分桶, 且排序的)
可以使用distribute by(id) sort by(id asc)  或是排序和分桶的字段相同的时候使用Cluster by(字段)
注意使用cluster by  就等同于分桶+排序(sort)
```



```sql
insert into table person_buck select id,name,sex,age from student distribute by(id) sort by(id asc);
```







## 分桶模式的参数

设置变量,设置分桶为true, 设置reduce数量是分桶的数量个数

set hive.enforce.bucketing = true;

不设置reduce的数量会使用默认的数量，默认的数量会和分桶的数量不一致，则不能分出正确分桶

set mapreduce.job.reduces=4;

本地模式

set hive.exec.mode.local.auto=true

动态分区

--设置为true表示开启动态分区功能（默认为false）

set hive.exec.dynamic.partition=true;

--设置为nonstrict,表示允许所有分区都是动态的（默认为strict）

set hive.exec.dynamic.partition.mode=nonstrict;



## Update与分桶表关系

Hive对使用Update功能的表有特定的语法要求, 语法要求如下:
(1)要执行Update的表中, 建表时必须带有buckets(分桶)属性
(2)要执行Update的表中, 需要指定格式,其余格式目前赞不支持, 如:parquet格式, 目前只支持ORCFileformat和AcidOutputFormat
(3)要执行Update的表中, 建表时必须指定参数('transactional' = true);
举例:

```
create table student (id bigint,name string) clustered by (name) into 2 buckets stored as orc TBLPROPERTIES('transactional'='true');
```


## 更新语句:

```
update student set id='444' where name='tom';
```



# 分桶表测试

这个例子就是将分区的字段进行hash散列将数据分桶到分桶数个文件中去

导入一个文件到分桶表里面

##  创建表

```sql
create table t_buk(id int,name string) clustered by(id)  sorted by(id DESC) into 4 buckets row format delimited``fields terminated by ',';
```

## 创建数据

**cd /usr/hive/hivedata/**

**vim buk.txt** 

```
1,数据库
2,数学
3,信息系统
4,操作系统
5,数据结构
6,数据no
7,数据other
8,数据time
9,数据操作
10,数据挖掘
11,数据挖机
12,数据信号
```



## 读取本地文件

```
load data local inpath '/usr/hive/hivedata/buk.txt' into table t_buk;
```

load方式这样导入数据到一个分桶表里面，是不会作出分桶的操作的，不会分成桶数个文件，还是一个文件在hdfs系统中

# 注意

1. 要想导入到数据到分桶表里面，必须是一个是已经是分桶的数据，比如已经形成了分桶数据个文件，才可以导入到分桶表里面，导数据的时候是不会将原来的数据形式变成分桶的数据形式

# hive分桶表的使用场景

所以一般是在一个表中查询了数据然后在塞入到一个分区表里面，查询是走mapReduce程序，然后将数据按分桶表照分桶的策略写入到分桶表中

形如

```sql
insert into t_buk select * from other … …;
```

后面的

清除数据

```sql
truncate table t_buk;
```

创建一个表来读取数据

```sql
create table t_p(id int,name string)
row format delimited
fields terminated by ',';

load data local inpath '/usr/hive/hivedata/buk.txt' into table t_p;
```

 insert into table t_buk select id,name from t_p; 

 insert overwirte 也可以

## 结果如下

```
Number of reduce tasks is set to 0 since there's no reduce operator
INFO  : number of splits:1
INFO  : Submitting tokens for job: job_1502537431423_0011
INFO  : The url to track the job: http://bigdata1:8088/proxy/application_1502537431423_0011/
INFO  : Starting Job = job_1502537431423_0011, Tracking URL = http://bigdata1:8088/proxy/application_1502537431423_0011/
INFO  : Kill Command = /home/bigdata/apps/hadoop/bin/hadoop job  -kill job_1502537431423_0011
INFO  : Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
INFO  : 2017-08-15 21:12:15,669 Stage-1 map = 0%,  reduce = 0%
INFO  : 2017-08-15 21:12:32,386 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 1.27 sec
INFO  : MapReduce Total cumulative CPU time: 1 seconds 270 msec
INFO  : Ended Job = job_1502537431423_0011
INFO  : Stage-4 is selected by condition resolver.
INFO  : Stage-3 is filtered out by condition resolver.
INFO  : Stage-5 is filtered out by condition resolver.
INFO  : Moving data to: hdfs://bigdata1:9000/user/hive/warehouse/t_buk/.hive-staging_hive_2017-08-15_21-12-02_490_4088487413275551800-3/-ext-10000 from hdfs://bigdata1:9000/user/hive/warehouse/t_buk/.hive-staging_hive_2017-08-15_21-12-02_490_4088487413275551800-3/-ext-10002
INFO  : Loading data to table default.t_buk from hdfs://bigdata1:9000/user/hive/warehouse/t_buk/.hive-staging_hive_2017-08-15_21-12-02_490_4088487413275551800-3/-ext-10000
INFO  : Table default.t_buk stats: [numFiles=1, numRows=12, totalSize=167, rawDataSize=155]
```

查看hdfs管理页面 50070

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fu7ufetxnpj30yw06fmx9.jpg)

 还是只有一文件，表示分桶不成功，没设reduce数量，使用默认的数量1，和我们期望分桶数量不一致

# 设置分桶参数

因为没有启动模式的开关，如下

设置变量,设置分桶为true, 设置reduce数量是分桶的数量个数

```
set hive.enforce.bucketing = true;
set mapreduce.job.reduces=4;
set hive.exec.mode.local.auto=true;
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
```

## 重新创建表



可以通过set hive.enforce.bucketing查看是否设置成功

先查看sort by (id)；

根据4个reduce来局部有序，每个reduce有序，但是从哪儿截断每个reduce并不确定

```
select id,name from t_p sort by (id);
INFO  : Number of reduce tasks not specified. Defaulting to jobconf value of: 4
INFO  : In order to change the average load for a reducer (in bytes):
INFO  :   set hive.exec.reducers.bytes.per.reducer=<number>
INFO  : In order to limit the maximum number of reducers:
INFO  :   set hive.exec.reducers.max=<number>
INFO  : In order to set a constant number of reducers:
INFO  :   set mapreduce.job.reduces=<number>
INFO  : number of splits:1
INFO  : Submitting tokens for job: job_1502537431423_0012
INFO  : The url to track the job: http://bigdata1:8088/proxy/application_1502537431423_0012/
INFO  : Starting Job = job_1502537431423_0012, Tracking URL = http://bigdata1:8088/proxy/application_1502537431423_0012/
INFO  : Kill Command = /home/bigdata/apps/hadoop/bin/hadoop job  -kill job_1502537431423_0012
INFO  : Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 4
INFO  : 2017-08-15 21:26:14,284 Stage-1 map = 0%,  reduce = 0%
INFO  : 2017-08-15 21:26:24,633 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 2.29 sec
INFO  : 2017-08-15 21:26:38,745 Stage-1 map = 100%,  reduce = 50%, Cumulative CPU 6.99 sec
INFO  : 2017-08-15 21:26:43,887 Stage-1 map = 100%,  reduce = 67%, Cumulative CPU 6.99 sec
INFO  : 2017-08-15 21:26:46,970 Stage-1 map = 100%,  reduce = 75%, Cumulative CPU 9.06 sec
INFO  : 2017-08-15 21:26:50,081 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 10.8 sec
INFO  : MapReduce Total cumulative CPU time: 10 seconds 800 msec
INFO  : Ended Job = job_1502537431423_0012
+-----+----------+--+
| id  |   name   |
+-----+----------+--+
| 4   | 操作系统     |
| 8   | 数据time   |
| 12  | 数据信号     |
| 2   | 数学       |
| 6   | 数据no     |
| 1   | 数据库      |
| 3   | 信息系统     |
| 5   | 数据结构     |
| 10  | 数据挖掘     |
| 11  | 数据挖机     |
| 7   | 数据other  |
| 9   | 数据操作     |
+-----+----------+--+
```



再试一次select 插入（将t_buk truncate也可，也可使用overwrite关键字）

 `insert overwrite table t_buk select id,name from t_p cluster by (id);`

再查看hdfs ui页面50070

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fu7uk05fz5j30x70a8mxm.jpg)

## 再分别查看这几个文件

```
$HADOOP_HOME/bin/hadoop fs -cat /user/hive/warehouse/t_buk/000000_0
$HADOOP_HOME/bin/hadoop fs -cat /user/hive/warehouse/t_buk/000001_0  
$HADOOP_HOME/bin/hadoop fs -cat /user/hive/warehouse/t_buk/000002_0  
$HADOOP_HOME/bin/hadoop fs -cat /user/hive/warehouse/t_buk/000003_0
```

## 得到结果

```
[bigdata@master hivedata]$  hadoop fs -cat /user/hive/warehouse/t_buk/000000_0  
4,操作系统
8,数据time
12,数据信号
[bigdata@master hivedata]$  hadoop fs -cat /user/hive/warehouse/t_buk/000001_0  
1,数据库
5,数据结构
9,数据操作
[bigdata@master hivedata]$  hadoop fs -cat /user/hive/warehouse/t_buk/000002_0  
2,数学
6,数据no
10,数据挖掘
[bigdata@master hivedata]$  hadoop fs -cat /user/hive/warehouse/t_buk/000003_0
3,信息系统
7,数据other
11,数据挖机
```

# 分桶表疑问

## 为什么每个桶里面的数据条数不一样

```
hash散列的时候数据可能将数据有的分的多，有的分的少
```

cluster by （id） 根据id分桶，桶内根据id排序，相当于 distribute by 和 sort by的集合，只是指定的字段都是同一个
用两个组合更加强大，分桶字段排序字段可以设置为不同

# 分桶表的意义：

提高join操作的效率案例

> 如果a表和b表已经是分桶表，而且分桶的字段都是是id字段
> 做这个join操作是，还需要做笛卡尔积吗？ 这样不需要，因为同一id哈希后的数据是一致的，这就是分桶表存在的意义

# 注意

1. 在分桶表中使用order by 是非常不建议的，这样会设置成一个reduce，强行将数据写入，一个reduce的内存会爆炸
2. 使用cluster by  就等同于分桶+排序(sort) 

 insert overwrite table student_buck  select * from student cluster by(Sno) sort by(Sage);  报错,cluster 和 sort 不能共存


---
title: Hive建表及sql相关
date: 2018年08月06日 22时15分52秒
tags: [Hive,使用]
categories: 大数据
toc: true
---

[TOC]

![](https://ws1.sinaimg.cn/large/0069RVTdgy1fu5k3varq8j31880g8js9.jpg)



hive主要是做离线日志分析的，不是为了做单行的事务控制的数据
新版hive也支持单行数据的读取，但是效率非常低，所以也没有什么updata语句

<!-- more -->


hdfs的数据是放在hdfs里面的，表的描述的结构元数据信息是放在mysql里面
hdfs中数据的信息在以下类似目录
**/user/hive/warehouse/thishive.db/book/country=japan**

可以在hive的客户端直接敲用hdfs的命令查看到
```
hdfs dfs -ls /hive目录
```

## 本地模式

set hive.exec.mode.local.auto=true;

## 建表(默认是内部表)

```sql
create table inner_table(id bigint, account string, income double, expenses double, time string) row format delimited fields terminated by '\t';
```

### 建分区表
```sql
create table outter_table(id bigint, account string, income double, expenses double, time string) partitioned by (logdate string) row format delimited fields terminated by '\t';
```

### 建外部表

```sql
create external table td_ext(id bigint, account string, income double, expenses double, time string) row format delimited fields terminated by '\t' location '/td_ext';
```

localtion是表示存放的位置

## 复制表

```
create  table 表1 like 表2 ;
```

将表2的结构复制到表1
将文件的数据导入到表中

## 导入数据到表中

   给表导入数据（若是分区表，则导入的时候需要加partition(#####)）

```bash
load data local inpath '/home/hadoop/mylog.log' into table 表名 partition(datestr='2013-09-18') ;
```

如果是导入本地文件，需要加参数local，如果是hdfs上的话，则不加

## 导出hive表中数据

```
insert overwrite local directory '/home/hadoop/student.txt'  select * from 表名;
```

> 不加local表示导出到hdfs

## 外部表和内部表的区别

drop table 外部表； 只会将外部表的结构 元数据信息删除，而不会删除外表的数据
drop table 内部表；会将内部表的结构元数据信息及其数据信息全部删除

# 保存select查询结果的几种方式：

```
1、将查询结果保存到一张新的hive表中

create table t_tmp
as
select * from t_p;

2、将查询结果保存到一张已经存在的hive表中

insert into  table t_tmp 
select * from t_p;

3、将查询结果保存到指定的文件目录（可以是本地，也可以是hdfs）

本地
insert overwrite local directory '/home/hadoop/student.txt'

select * from student;

导入到mysql
insert overwrite directory '/aaa/test'
select * from t_p;



```

## 分区表

普通表和分区表区别：有大量数据增加的需要建分区表
分区的字段会自动加在表结构上

这个是将导入到fruit的分区里面

```bash
load data local inpath '/home/bigdata/food.txt' overwrite into table book partition (type='fruit')；
```


在hdfs里面，分区表会存在多个不同的目录，但是在查询的时候，还是将多个分区表的信息融入到一个表中

## 使用

如果是使用overwrite命令，必须加stored as textfile；

### 小操作

以下资源来自网络（若有不合适，请联系我）

### students.txt

```
95001,李勇,男,20,CS
95002,刘晨,女,19,IS
95003,王敏,女,22,MA
95004,张立,男,19,IS
95005,刘刚,男,18,MA
95006,孙庆,男,23,CS
95007,易思玲,女,19,MA
95008,李娜,女,18,CS
95009,梦圆圆,女,18,MA
95010,孔小涛,男,19,CS
95011,包小柏,男,18,MA
95012,孙花,女,20,CS
95013,冯伟,男,21,CS
95014,王小丽,女,19,CS
95015,王君,男,18,MA
95016,钱国,男,21,MA
95017,王风娟,女,18,IS
95018,王一,女,19,IS
95019,邢小丽,女,19,IS
95020,赵钱,男,21,IS
95021,周二,男,17,MA
95022,郑明,男,20,MA
```

### sc.txt

```
95001,1,81
95001,2,85
95001,3,88
95001,4,70
95002,2,90
95002,3,80
95002,4,71
95002,5,60
95003,1,82
95003,3,90
95003,5,100
95004,1,80
95004,2,92
95004,4,91
95004,5,70
95005,1,70
95005,2,92
95005,3,99
95005,6,87
95006,1,72
95006,2,62
95006,3,100
95006,4,59
95006,5,60
95006,6,98
95007,3,68
95007,4,91
95007,5,94
95007,6,78
95008,1,98
95008,3,89
95008,6,91
95009,2,81
95009,4,89
95009,6,100
95010,2,98
95010,5,90
95010,6,80
95011,1,81
95011,2,91
95011,3,81
95011,4,86
95012,1,81
95012,3,78
95012,4,85
95012,6,98
95013,1,98
95013,2,58
95013,4,88
95013,5,93
95014,1,91
95014,2,100
95014,4,98
95015,1,91
95015,3,59
95015,4,100
95015,6,95
95016,1,92
95016,2,99
95016,4,82
95017,4,82
95017,5,100
95017,6,58
95018,1,95
95018,2,100
95018,3,67
95018,4,78
95019,1,77
95019,2,90
95019,3,91
95019,4,67
95019,5,87
95020,1,66
95020,2,99
95020,5,93
95021,2,93
95021,5,91
95021,6,99
95022,3,69
95022,4,93
95022,5,82
95022,6,100
```



### course.txt

```
1,数据库
2,数学
3,信息系统
4,操作系统
5,数据结构
6,数据处理
```

### 建表

```sql
create table student(Sno int,Sname string,Sex string,Sage int,Sdept string)row format delimited fields terminated by ','stored as textfile;
create table course(Cno int,Cname string) row format delimited fields terminated by ',' stored as textfile;
create table sc(Sno int,Cno int,Grade int)row format delimited fields terminated by ',' stored as textfile;

load data local inpath '/home/bigdata/apps/hive/hivedata/students.txt' overwrite into table student;
load data local inpath '/home/bigdata/apps/hive/hivedata/sc.txt' overwrite into table sc;
load data local inpath '/home/bigdata/apps/hive/hivedata/course.txt' overwrite into table course;
```
### sql需求

```sql
查询全体学生的学号与姓名
　　hive> select Sno,Sname from student;

查询选修了课程的学生姓名
　　hive> select distinct Sname from student inner join sc on student.Sno=Sc.Sno;

----hive的group by 和集合函数

查询学生的总人数
　　hive> select count(distinct Sno)count from student;

计算1号课程的学生平均成绩
　　hive> select avg(distinct Grade) from sc where Cno=1;
查询各科成绩平均分
		hive> select Cno,avg(Grade) from sc group by Cno;  
查询选修1号课程的学生最高分数
　　select Grade from sc where Cno=1 sort by Grade desc limit 1; 
(注意比较:select * from sc where Cno=1 sort by Grade
		  select Grade from sc where Cno=1 order by Grade)     
　　   
　　
求各个课程号及相应的选课人数 
　　hive> select Cno,count(1) from sc group by Cno;


查询选修了3门以上的课程的学生学号
　　hive> select Sno from (select Sno,count(Cno) CountCno from sc group by Sno)a where a.CountCno>3;
或　hive> select Sno from sc group by Sno having count(Cno)>3; 

----hive的Order By/Sort By/Distribute By
　　Order By ，在strict 模式下（hive.mapred.mode=strict),order by 语句必须跟着limit语句，但是在nonstrict下就不是必须的，这样做的理由是必须有一个reduce对最终的结果进行排序，如果最后输出的行数过多，一个reduce需要花费很长的时间。

查询学生信息，结果按学号全局有序
　　hive> set hive.mapred.mode=strict;   <默认nonstrict>
hive> select Sno from student order by Sno;
FAILED: Error in semantic analysis: 1:33 In strict mode, if ORDER BY is specified, LIMIT must also be specified. Error encountered near token 'Sno'
　　Sort By，它通常发生在每一个redcue里，“order by” 和“sort by”的区别在于，前者能给保证输出都是有顺序的，而后者如果有多个reduce的时候只是保证了输出的部分有序。set mapred.reduce.tasks=<number>在sort by可以指定，在用sort by的时候，如果没有指定列，它会随机的分配到不同的reduce里去。distribute by 按照指定的字段对数据进行划分到不同的输出reduce中 
　　此方法会根据性别划分到不同的reduce中 ，然后按年龄排序并输出到不同的文件中。

查询学生信息，按性别分区，在分区内按年龄有序
　　hive> set mapred.reduce.tasks=2;
　　hive> insert overwrite local directory '/home/hadoop/out' 
select * from student distribute by Sex sort by Sage;

----Join查询,join只支持等值连接 
查询每个学生及其选修课程的情况
　　hive> select student.*,sc.* from student join sc on (student.Sno =sc.Sno);
查询学生的得分情况。
　　hive>select student.Sname,course.Cname,sc.Grade from student join sc on student.Sno=sc.Sno join course on sc.cno=course.cno;

查询选修2号课程且成绩在90分以上的所有学生。
　　hive> select student.Sname,sc.Grade from student join sc on student.Sno=sc.Sno 
where  sc.Cno=2 and sc.Grade>90;
　　
----LEFT，RIGHT 和 FULL OUTER JOIN ,inner join, left semi join
查询所有学生的信息，如果在成绩表中有成绩，则输出成绩表中的课程号
　　hive> select student.Sname,sc.Cno from student left outer join sc on student.Sno=sc.Sno;
　　如果student的sno值对应的sc在中没有值，则会输出student.Sname null.如果用right out join会保留右边的值，左边的为null。
　　Join 发生在WHERE 子句之前。如果你想限制 join 的输出，应该在 WHERE 子句中写过滤条件——或是在join 子句中写。
　　
----LEFT SEMI JOIN  Hive 当前没有实现 IN/EXISTS 子查询，可以用 LEFT SEMI JOIN 重写子查询语句

重写以下子查询为LEFT SEMI JOIN
  SELECT a.key, a.value
  FROM a
  WHERE a.key exist in
   (SELECT b.key
    FROM B);
可以被重写为：
   SELECT a.key, a.val
   FROM a LEFT SEMI JOIN b on (a.key = b.key)

查询与“刘晨”在同一个系学习的学生
　　hive> select s1.Sname from student s1 left semi join student s2 on s1.Sdept=s2.Sdept and s2.Sname='刘晨';

注意比较：
select * from student s1 left join student s2 on s1.Sdept=s2.Sdept and s2.Sname='刘晨';
select * from student s1 right join student s2 on s1.Sdept=s2.Sdept and s2.Sname='刘晨';
select * from student s1 inner join student s2 on s1.Sdept=s2.Sdept and s2.Sname='刘晨';
select * from student s1 left semi join student s2 on s1.Sdept=s2.Sdept and s2.Sname='刘晨';
select s1.Sname from student s1 right semi join student s2 on s1.Sdept=s2.Sdept and s2.Sname='刘晨';
```


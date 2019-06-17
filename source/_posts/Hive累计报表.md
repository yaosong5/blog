---
title: Hive累计报表
date: 2018年08月06日 22时15分52秒
tags: [Hive,报表]
categories: 大数据
toc: true
---

[TOC]

在 hive 做统计的时候，总是涉及到做累计的报表处理，下面案列就是来做相应处理

# 准备

## 创建数据文件

（在hadoop所在的机器）

**vim /usr/hadoop/hivedata/t_sales.dat**

```
舒肤佳,2018-06,5
舒肤佳,2018-06,15
美姿,2018-06,5
舒肤佳,2018-06,8
美姿,2018-06,25
舒肤佳,2018-06,5
舒肤佳,2018-07,4
舒肤佳,2018-07,6
美姿,2018-07,10
美姿,2018-07,5
```

上传到hdfs

```shell
hadoop fs -put /usr/hadoop/hivedata/t_sales.dat /local/hivedata/t_sales.dat
```

<!-- more -->

## 创建表及读取数据

```sql
create table t_sales(brandname string,month string,sales int)
row format delimited fields terminated by ',';
load data inpath '/local/hivedata/t_sales.dat' into table t_sales;
```

如果是上传本地文件（如果在hive所在主机上） 则在load  data 后加 local，如 `load data local inpath '/usr/hadoop/hivedata/t_sales.dat' into table t_sales;`



# 1、先求每个品牌的月总金额

```
select brandname,month,sum(sales) as all_sales from t_sales group by brandname,month
```

![](https://img.gangtieguo.cn/0069RVTdgy1fu87l2nswej31500smq3k.jpg)



# 2、将月总金额自连接

```sql
select * from  (select brandname,month,sum(sales) as sal from t_sales group by brandname,month) A 
    inner join 
   (select brandname,month,sum(sales) as sal from t_sales group by brandname,month) B
    on
A.brandname=B.brandname
where 
B.month <= A.month;
```

![](https://img.gangtieguo.cn/006tNbRwgy1fu88kcmiosj31ik0zgq4y.jpg)





# 3、从上一步的结果中进行分组查询

分组的字段是 a.brandname a.month

求月累计值： 将 b.month <= a.month 的所有 b.sals求和即可

```
select A.brandname,A.month,max(A.sales) as sales,sum(B.sales) as accumulate
from 
(select brandname,month,sum(sales) as sales from t_sales group by brandname,month) A 
inner join 
(select brandname,month,sum(sales) as sales from t_sales group by brandname,month) B
on
A.brandname=B.brandname
where B.month <= A.month
group by A.brandname,A.month
order by A.brandname,A.month;
```

![](https://img.gangtieguo.cn/006tNbRwgy1fu88tmmjfqj31ag0vamyy.jpg)
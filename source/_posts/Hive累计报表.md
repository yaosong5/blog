---
title: Hive累计报表
date: 2018年08月06日 22时15分52秒
tags: [Hive,报表]
categories: 大数据
toc: true
---

[TOC]

在hive做统计的时候，总是涉及到做累计的报表处理，下面案列就是来做相应处理

# 准备

## 创建数据文件

**vim /usr/hive/hivedata/t_sales.dat**

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

<!-- more -->

## 创建表及读取数据

```sql
create table t_sales(brandname string,month string,sales int)
row format delimited fields terminated by ',';
load data local inpath '/usr/hive/hivedata/t_sales.dat' into table t_access_times;
```



# 1、先求个用户的月总金额

```sql
select brandname,month,sum(sales) as all_sales from t_sales group by brandname,month
```



# 2、将自连接

# 3、

进行分组查询，分组的字段是a.name a.month

求月累计值：  将b.month <= a.month的所有b.sales求和即可

```
select A.brandname,A.month,max(A.sales) as sales,sum(B.sales) as accumulate
from 
(select brandname,month,sum(sales) as sales from t_access_times group by brandname,month) A 

inner join 
(select brandname,month,sum(sales) as sales from t_access_times group by brandname,month) B
on

A.brandname=B.brandname

where B.month <= A.month

group by A.brandname,A.month

order by A.brandname,A.month;
```


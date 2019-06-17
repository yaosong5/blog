---
title: Hive解析任务-将json的多个属性拆分成多条记录
date:
tags: [Hive,数仓]
categories: 实际问题
toc: true
---

[TOC]

需求环境：

在hive表dwb.dwb_r_thrid_data中，data字段存放有json字符串

![](http://img.gangtieguo.cn/006tKfTcgy1g10yhuqk6bj32jc0e4wfy.jpg)

<!-- more -->

需要从json字符串中，解析到需要的字段：将一个json里面的属性data.loanInfo.mobile.timeScopes.D360、data.loanInfo.mobile.timeScopes.D90所包含的字段分别解析成一条记录，并且将D360、D90也作为字段timeScope的值解析到该条记录中。

```json
{
    "code":"0",
    "data":{
        "loanInfo":{
            "mobile":{
                "timeScopes":{
                    "D360":{
                        "maxOverdueDays":-1,
                        "loanTenantCount":0,
                        "monthsFromFirstLoan":-1,
                        "averageLoanGapDays":-1,
                        "averageLoanAmount":0,
                        "averageTenantGapDays":-1,
                        "loanCount":0,
                        "maxLoanAmount":0,
                        "daysFromLastLoan":-1,
                        "overdueTenantCount":-1,
                        "queryCount":0,
                        "monthsFromLastOverdue":-1,
                        "maxLoanPeriodDays":0,
                        "remainingAmount":-1,
                        "monthsForNormalRepay":-1,
                        "overdueLoanCount":-1,
                        "overdueFor2TermTenantCount":-1
                    },
                    "D90":{
                        "maxOverdueDays":-1,
                        "loanTenantCount":0,
                        "averageLoanGapDays":-1,
                        "averageLoanAmount":0,
                        "averageTenantGapDays":-1,
                        "overdueLoanCount":-1,
                        "overdueFor2TermTenantCount":-1,
                        "loanCount":0,
                        "maxLoanAmount":0,
                        "overdueTenantCount":-1,
                        "queryCount":0,
                        "maxLoanPeriodDays":0
                    }
                }
            },
            "pid":{
                "timeScopes":{
                    "D360":{
                        "maxOverdueDays":-1,
                        "loanTenantCount":0,
                        "monthsFromFirstLoan":-1,
                        "averageLoanGapDays":-1,
                        "averageLoanAmount":0,
                        "averageTenantGapDays":-1,
                        "loanCount":0,
                        "maxLoanAmount":0,
                        "daysFromLastLoan":-1,
                        "overdueTenantCount":-1,
                        "queryCount":0,
                        "monthsFromLastOverdue":-1,
                        "maxLoanPeriodDays":0,
                        "remainingAmount":-1,
                        "monthsForNormalRepay":-1,
                        "overdueLoanCount":-1,
                        "overdueFor2TermTenantCount":-1
                    },
                    "D90":{
                        "maxOverdueDays":-1,
                        "loanTenantCount":0,
                        "averageLoanGapDays":-1,
                        "averageLoanAmount":0,
                        "averageTenantGapDays":-1,
                        "overdueLoanCount":-1,
                        "overdueFor2TermTenantCount":-1,
                        "loanCount":0,
                        "maxLoanAmount":0,
                        "overdueTenantCount":-1,
                        "queryCount":0,
                        "maxLoanPeriodDays":0
                    }
                }
            },
            "deviceId":{
                "timeScopes":{
                    "D360":{
                        "loanTenantCount":0,
                        "loanCount":0,
                        "queryCount":0
                    },
                    "D90":{
                        "loanTenantCount":0,
                        "loanCount":0,
                        "queryCount":0
                    }
                }
            }
        },
        "blacklist":{
            "mobile":{
                "lastConfirmAtDays":-1,
                "lastConfirmStatus":"",
                "blackLevel":"none",
                "last6MTenantCount":0,
                "last6MQueryCount":0,
                "last12MMaxConfirmStatus":""
            },
            "pid":{
                "lastConfirmAtDays":-1,
                "lastConfirmStatus":"",
                "blackLevel":"none",
                "last6MTenantCount":0,
                "last6MQueryCount":0,
                "last12MMaxConfirmStatus":""
            },
            "deviceId":{
                "lastConfirmAtDays":-1,
                "lastConfirmStatus":"",
                "blackLevel":"none",
                "last6MTenantCount":0,
                "last6MQueryCount":0,
                "last12MMaxConfirmStatus":""
            }
        }
    },
    "message":"请求成功"
}
```

表结构形如：

```sql
create table if not exists dwb.dwb_r_morpho_loaninfo_mobile(
       apply_risk_id                        string comment  "风控ID",
       dp_data_id                           string comment  "dp_dataID",
       maxOverdueDays                       string,
       loanTenantCount                      string,
       monthsFromFirstLoan                  string,
       averageLoanGapDays                   string,
       averageLoanAmount                    string,
       averageTenantGapDays                 string,
       loanCount                            string,
       maxLoanAmount                        string,
       daysFromLastLoan                     string,
       overdueTenantCount                   string,
       queryCount                           string,
       monthsFromLastOverdue                string,
       maxLoanPeriodDays                    string,
       remainingAmount                      string,
       monthsForNormalRepay                 string,
       overdueLoanCount                     string,
       overdueFor2TermTenantCount           string,
       timeScope                            string comment  "时间期限",
       morpho_created_at                    string comment  "创建时间",
       etl_time                             string comment  "etl处理时间"
) comment 'moblie' 
 PARTITIONED BY (dt string  comment '分区日期')
 row format delimited fields terminated by '\001'
 NULL DEFINED AS ''
 stored as orc;

```

接下来就开始表演吧。

# 如果是json数组,可以很方便拆分

我们都知道对于一条json里面值为json数组的属性，hive可以将其获取到并且进行拆分成多条记录：

如以下infoquerybean属性：

```Json
{
    "overduemoreamt":"0",
    "loancount":"0",
    "loanbal":"0",
    "outstandcount":"0",
    "queryatotalorg":"最近***********",
    "loanamt":"0",
    "overdueamt":"0",
    "generationcount":"0",
    "msgContent":"成功!",
    "generationamount":"0",
    "overduemorecount":"0",
    "totalorg":"*************",
    "infoquerybean":[
        {
            "s_value":"审批",
            "ddate":"2018-09-10",
            "ordernum":"1"
        },
        {
            "s_value":"审批",
            "ddate":"2018-09-06",
            "ordernum":"2"
        },
        {
            "s_value":"审批",
            "ddate":"2018-08-21",
            "ordernum":"3"
        },
        {
            "s_value":"审批",
            "ddate":"2018-08-09",
            "ordernum":"4"
        },
        {
            "s_value":"审批",
            "ddate":"2018-07-28",
            "ordernum":"5"
        },
        {
            "s_value":"审批",
            "ddate":"2018-07-27",
            "ordernum":"6"
        }
    ],
    "overduecount":"0",
    "msgCode":"200"
}
```

可以通过split拆分成结果,插入到形状如下的表中：

```sql
create table if not exists dwb.dwb_r_nifa_share_detail_n(
       apply_risk_id                        string comment  "风控ID",
       dp_data_id                           string comment  "dp_dataID",
       nifa_share_detail_ordernum string comment '序号',
       nifa_share_detail_ddate string comment '查询日期',
       nifa_share_detail_s_value string comment '查询原因',
       nifa_share_created_at                string comment  "创建时间",
       etl_time                             string comment  "etl处理时间"
) comment 'table test' 
 PARTITIONED BY (dt string  comment '分区日期')
 row format delimited fields terminated by '\001'
 NULL DEFINED AS ''
 stored as orc;
```

通过 

```sql
explode(split(default.get_json_path(a.data,'infoquerybean'),'@\\|@'))
```

将其拆分成多条记录，完整sql见下：

```Sql
dt=$1
hive<<!
set mapreduce.job.queuename=root.dw;
set hive.support.concurrency=false;

insert overwrite table dwb.dwb_r_nifa_share_detail_n partition(dt='$dt')
select 
      a.apply_risk_id,
      a.dp_data_id,
      nifa_share_detail_ordernum,
      nifa_share_detail_ddate,
      nifa_share_detail_s_value,
      from_unixtime(cast(a.timestamp/1000 as bigint),'yyyy-MM-dd HH:mm:ss') as nifa_share_created_at,
      current_timestamp() as etl_time
from (select dwb_r_thrid_data.apply_risk_id,dwb_r_thrid_data.dp_data_id,dwb_r_thrid_data.data,dwb_r_thrid_data.timestamp 
       from dwb.dwb_r_thrid_data   where  channel_name = 'nifa_prod' and interface_name = 'share' and get_json_object(data,'$.msgCode') = '200' and dwb_r_thrid_data.dt='$dt'
 ) a 
 lateral view explode(split(default.get_json_path(a.data,'infoquerybean'),'@\\\\|@')) b as infoquerybean
 lateral view default.json_tuple2(b.infoquerybean,'ordernum','ddate','ordernum') c as nifa_share_detail_ordernu,nifa_share_detail_ddate, nifa_share_detail_s_value
;
!
```

得到结果：

![得到结果](http://img.gangtieguo.cn/006tKfTcgy1g1100n1r1pj32s20d8t9o.jpg)



# 顺着json数组思路，改造json样式

通过get_json_object()方法，得到两个json属性，通过concat拼接成json数组，就可以像上面那样拆分成多条记录。（测试阶段的样例都使用了设定分区dt,限制条数，因为这样测试起来很快，只需要三秒!!!!😁😁😁😁）



## 1.通过get_json_object方法

```Sql
select get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D360") d3,get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D90")  d9  from dwb.dwb_r_thrid_data td  where channel_name ='morpho' and interface_name ='query' and dt='20190218' limit 5
```

得到

![](http://img.gangtieguo.cn/006tKfTcgy1g110nazcuwj329c032aa3.jpg)

## 2.拼接获取的D360和D90字段

```sql
select td.*,concat(regexp_replace(get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D360"),'}',',"timeScope":"D360"}'),"|",regexp_replace(get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D90"),'}',',"timeScope":"D90"}')) ts  from dwb.dwb_r_thrid_data td  where  channel_name ='morpho' and interface_name ='query' and dt='20190218' limit 5
```

![](http://img.gangtieguo.cn/006tKfTcgy1g110qkjxjij31yk03gmx6.jpg)

拼接的字符串样式，通过"|"分隔两个对象 

```json
{"maxOverdueDays":-1,"monthsFromFirstLoan":-1,"loanTenantCount":0,"averageLoanGapDays":-1,"averageTenantGapDays":-1,"averageLoanAmount":0,"loanCount":0,"maxLoanAmount":0,"overdueTenantCount":-1,"daysFromLastLoan":-1,"queryCount":0,"monthsFromLastOverdue":-1,"maxLoanPeriodDays":0,"remainingAmount":-1,"monthsForNormalRepay":-1,"overdueLoanCount":-1,"overdueFor2TermTenantCount":-1,"timeScope":"D360"}|{"maxOverdueDays":-1,"loanTenantCount":0,"averageLoanGapDays":-1,"averageTenantGapDays":-1,"averageLoanAmount":0,"overdueLoanCount":-1,"overdueFor2TermTenantCount":-1,"loanCount":0,"overdueTenantCount":-1,"maxLoanAmount":0,"queryCount":0,"maxLoanPeriodDays":0,"timeScope":"D90"}

```

## 3. 最终通过分隔符进行切分

对于涉及到分隔符，转义字符的个数，请参考该文章[数仓-解决hive处理异常json命令行转义字符的问题](https://yaosong5.gitbook.io/yao/shi-ji-wen-ti-jie-jue/shu-cang-jie-jue-hive-chu-li-yi-chang-json-ming-ling-hang-zhuan-yi-zi-fu-de-wen-ti)

```sql
select 
      a.apply_risk_id,
      a.dp_data_id,
      c.*,
      from_unixtime(cast(a.timestamp/1000 as bigint),'yyyy-MM-dd HH:mm:ss') as morpho_created_at,
      current_timestamp() as etl_time
from (select td.*,concat(regexp_replace(get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D360"),'}',',"timeScope":"D360"}'),"|",regexp_replace(get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D90"),'}',',"timeScope":"D90"}')) ts
           from dwb.dwb_r_thrid_data td
           where
             channel_name ='morpho' and interface_name ='query' and td.dt='20190218' limit 5
     ) a
lateral view explode(split(a.ts,'\\\\|')) b as list
lateral view default.json_tuple2(b.list,'maxOverdueDays','loanTenantCount','monthsFromFirstLoan','averageLoanGapDays','averageLoanAmount','averageTenantGapDays','loanCount','maxLoanAmount','daysFromLastLoan','overdueTenantCount','queryCount','monthsFromLastOverdue','maxLoanPeriodDays','remainingAmount','monthsForNormalRepay','overdueLoanCount','overdueFor2TermTenantCount','timeScope') 
c as maxOverdueDays,loanTenantCount,monthsFromFirstLoan,averageLoanGapDays,averageLoanAmount,averageTenantGapDays,loanCount,maxLoanAmount,daysFromLastLoan,overdueTenantCount,queryCount,monthsFromLastOverdue,maxLoanPeriodDays,remainingAmount,monthsForNormalRepay,overdueLoanCount,overdueFor2TermTenantCount,timeScope
```

搞定

![得到结果](http://img.gangtieguo.cn/006tKfTcgy1g1115ip6uyj328m0bkjrw.jpg)



# 完整样例

```sql
dt=$1

hive<<!
set mapreduce.job.queuename=root.dw;
set hive.support.concurrency=false;

insert overwrite table dwb.dwb_r_morpho_loaninfo_mobile partition(dt='$dt')
select 
      a.apply_risk_id,
      a.dp_data_id,
      c.*,
      from_unixtime(cast(a.timestamp/1000 as bigint),'yyyy-MM-dd HH:mm:ss') as morpho_created_at,
      current_timestamp() as etl_time
from (select td.*,concat(regexp_replace(get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D360"),'}',',"timeScope":"D360"}'),"|",regexp_replace(get_json_object(td.data,"$.data.loanInfo.mobile.timeScopes.D90"),'}',',"timeScope":"D90"}')) ts
           from dwb.dwb_r_thrid_data td   where channel_name ='morpho' and interface_name ='query' 
     ) a
lateral view explode(split(a.ts,'\\\\|')) b as list
lateral view default.json_tuple2(b.list,'maxOverdueDays','loanTenantCount','monthsFromFirstLoan','averageLoanGapDays','averageLoanAmount','averageTenantGapDays','loanCount','maxLoanAmount','daysFromLastLoan','overdueTenantCount','queryCount','monthsFromLastOverdue','maxLoanPeriodDays','remainingAmount','monthsForNormalRepay','overdueLoanCount','overdueFor2TermTenantCount','timeScope') 
c as maxOverdueDays,loanTenantCount,monthsFromFirstLoan,averageLoanGapDays,averageLoanAmount,averageTenantGapDays,loanCount,maxLoanAmount,daysFromLastLoan,overdueTenantCount,queryCount,monthsFromLastOverdue,maxLoanPeriodDays,remainingAmount,monthsForNormalRepay,overdueLoanCount,overdueFor2TermTenantCount,timeScope
;
!
```







参考：[HIVE: lateral view explode & json_turpe 实现 json数组行转列&字段拆分](http://www.lilysui.cn/tech/2017/11/02/row-to-column)


---
title: Hbase-shell操作
date: 2018年08月06日 22时15分52秒
tags:  [HBase,Shell]
categories: 大数据
toc: true
---

[TOC]

hbase使用命令行操作，简单直接，方便快捷，掌握一点必备的基础命令。

HBase启动命令行

```bash
$HBASE_HOME/bin/hbase shell
```

<!-- more -->

## 创建表

```bash
create 'testtable',{NAME=>'cf',VERSIONS=>2},{NAME=>'cf2',VERSIONS=>2}
```

## 查看表结构

```bash
disable 'testtable'
```



## 删除表

```bash
drop 'testtable'
```

## 修改表

```bash
disable 'testtable'
alter 'testtable',{NAME=>'cf',TTL=>'10000000'},{NAME=>'cf2',TTL=>'10000000'}
enable 'testtable'
修改表必须先 disable 表
```

## 表数据的增删查改：

### 添加数据：

```Bash
put 'testtable','rowkey1','cf:key1','val1'
```



### 查询数据:

```bash
get 'testtable','rowkey1','cf:key1'
get 'testtable','rowkey1', {COLUMN=>'cf:key1'}
```



### 扫描表:

```bash
scan 'testtable',{COLUMNS=>cf:col1,LIMIT=>5} #可以添加STARTROW、TIMERANGE和FITLER等高级功能

```

### 查询表中的数据行数:

语法：count <table>, {INTERVAL => intervalNum, CACHE => cacheNum}

```Bash
count 'testtable',{INTERVAL => 100, CACHE => 500}
```



### 删除数据:

```bash
delete 'testtable','rowkey1','cf:key1'
truncate 'testtable'
```


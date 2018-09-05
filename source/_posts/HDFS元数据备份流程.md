---
title: HDFS元数据备份流程
date: 2018年08月06日 22时15分52秒
tags: [HDFS,原理,Hadoop]
categories: 大数据
toc: true
---

[TOC]

# hdfs元数据管理

namenode对数据的管理采用了三种存储形式：

- 内存元数据(NameSystem)

- 磁盘元数据镜像文件fsimage

- 数据操作日志文件edits（可通过日志运算出元数据）

  <!-- more -->![](http://pebgsxjpj.bkt.clouddn.com/15361544670231.jpg)

磁盘文件我们可以通过查看namenode的目录结构看到：
![](http://pebgsxjpj.bkt.clouddn.com/15361574063204.jpg)

A、内存中有一份完整的元数据(内存meta data)
B、磁盘有一个“准完整”的元数据镜像（fsimage）文件(在namenode的工作目录中)
C、用于衔接内存metadata和持久化元数据镜像fsimage之间的操作日志（edits文件
注：当客户端对hdfs中的文件进行新增或者修改操作，操作记录首先被记入edits日志文件中，当客户端操作成功后，相应的元数据会更新到内存meta.data中



# namenode管理元数据解析

namenode会将元数据放在内存里面，这样方便快速对数据的请求，由于放在内存中是不安全的，所以就序列化到fsimage里面，就像jvm中dump，将内存中所有数据dump出去,由于一条元数据大小为150byte，hdfs不利于存储小块文件，所以存大文件划算，因为元数据消耗的内存都是一样的，但是内存中的数据量太大，不可能经常序列化，所以需要定时序列化

## 所以引入了secondaryNameNode

更新元数据的时候，不可能去直接跟更改元数据fsimage文件，因为文件是线性结构，假如遇到更改中间内容会很不方便，所有就将操作信息记录在edits日志文件中，只是记录操作信息

## edits文件定期转为元数据

为了防止edits过多，导致在启动hdfs集群datanode的时候会很慢，因为需要将edits通过转化形成为元数据fsimage文件，所以应该定期将edits文件转换为fsimage元数据，然后将fsimage替换掉

## secondaryNameNode的出现

如果nameNode来做上面的edits转换为元数据的话，由于消耗的资源太大，就不能为其他比如从hdfs中读取数据服务提供资源，或者提供服务的效果不好 
所以这个时候就把合并操作交给secondNameNode来做 
这个过程叫做checkpoint 

**namenode和secondaryNameNode最好不要放在一台机器上**

宕机可能导致数据不能恢复 
测试环境或者学习环境可以弄在一台机器上



## 为了防止namenode宕机导致了数据丢失所作配置

可以在hdfs-site.xml文件中在多个机器上的目录来保存name的edits，fsimage文件

```xml
<property>
        <name>dfs.name.dir</name>
        <value>/home/bigdata/names1,/home/bigdata/names2</value>
</property>
```

配置的多个的话，会同时往这两个目录中写

如果不配置这个默认的目录是core-site.xml文件中配置的hadoop的临时文件 

```xml
<property> 
   <name>hadoop.tmp.dir</name> 
    <value>/home/bigdata/apps/hadoop-2.6.4/tmp</value> 
</property> 
```



## 
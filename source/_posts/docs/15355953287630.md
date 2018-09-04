---
title: Hdfs结构性能分析及读写流程
date: 2018年08月06日 22时15分52秒
tags: [HDFS,原理,Hadoop]
categories: 大数据
toc: true
---

[TOC]





<!-- more -->

# namenode和secondaryNameNode最好不要放在一台机器上

宕机可能导致数据不能恢复 
测试环境或者学习环境可以弄在一台机器上

# hdfs中namenode和datanode的初始化

hdfs会在配置文件中配置一个datanode的工作目录元数据 
查看目录结构 tree hddata/ 
![img](file:///var/folders/9p/dbpbsyq158n7y12303j34hxm0000gp/T/WizNote/3172cdcb-77fe-4f72-8882-12fe40ddef6a/index_files/4ee70fb7-cfdc-4010-a387-986e268b5a1e.jpg) 
datanode的工作目录是在datanode启动后初始化的 
而hadoop namenode format 只会初始name的工作目录，和datanode没有关系

# 把一个hdfs的一个节点加入到另一个集群

必须要将hdfs datanode的工作目录删除，不然持有上一个集群的datanode的工作目录，会认为是一个误操作，为了防止丢失数据，不会让其连接上

# 如果集群够大，上百台机器

那么在hdfs上面，是需要配置机架感知

# namenode管理元数据

namenode会将元数据放在内存里面，这样方便快速对数据的请求 
但是放在内存中是不安全的，所有就序列化到fsimage里面 
![img](file:///var/folders/9p/dbpbsyq158n7y12303j34hxm0000gp/T/WizNote/3172cdcb-77fe-4f72-8882-12fe40ddef6a/index_files/2d3df4da-f64b-4ccf-967b-9ade7c2ab46f.jpg) 
就像jvm中dump，将内存中所有数据dump出去 
假如 
![img](file:///var/folders/9p/dbpbsyq158n7y12303j34hxm0000gp/T/WizNote/3172cdcb-77fe-4f72-8882-12fe40ddef6a/index_files/8fa4e8c2-8b29-4809-8219-b383aa6d8664.jpg) 
所以存大文件划算，因为元数据消耗的内存都是一样的 
![img](file:///var/folders/9p/dbpbsyq158n7y12303j34hxm0000gp/T/WizNote/3172cdcb-77fe-4f72-8882-12fe40ddef6a/index_files/2a0de522-13be-4777-b2f4-68377648bbb4.jpg) 
但是内存中的数据量太大，不可能经常序列化，所以需要定时序列化

## 所以引入了secondaryNameNode

更新元数据的时候，不可能去直接跟更改元数据fsimage文件，因为文件是线性结构，假如遇到更改中间内容会很不方便，所有就将操作信息记录在edits日志文件中，只是记录操作信息

### edits文件定期转为元数据

为了防止edits过多，导致在启动hdfs集群datanode的时候会很慢，因为需要将edits通过转化形成为元数据fsimage文件，所以应该定期将edits文件转换为fsimage元数据，然后将fsimage替换掉

### secondaryNameNode的出现

如果nameNode来做上面的edits转换为元数据的话，由于消耗的资源太大，就不能为其他比如从hdfs中读取数据服务提供资源，或者提供服务的效果不好 
所以这个时候就把合并操作交给secondNameNode来做 
这个过程叫做checkpoint 
![img](file:///var/folders/9p/dbpbsyq158n7y12303j34hxm0000gp/T/WizNote/3172cdcb-77fe-4f72-8882-12fe40ddef6a/index_files/d6a7699e-3158-4cfc-ab83-9fec25286004.jpg)

## 为了防止namenode宕机导致了数据丢失

可以在hdfs-site.xml文件中在多个机器上的目录来保存name的edits，fsimage文件

 `<property>``    <name>dfs.name.dir</name>``    <value>/home/bigdata/names1,/home/bigdata/names2</value>``</property>`

配置的多个的话，会同时往这两个目录中写

如果不配置这个默认的目录是core-site.xml文件中配置的hadoop的临时文件 
 ` 
 <property> 
​       <name>hadoop.tmp.dir</name> 
​       <value>/home/bigdata/apps/hadoop-2.6.4/tmp</value> 
   </property> `

# Hdfs的结构

1.HDFS集群分为两大角色：NameNode、DataNode （secondary NameNode）
2.NameNode负责管理整个文件系统的元数据
记录文件在哪里
3.DataNode 负责管理用户的文件数据块
不负责切块，负责保管
4.文件会按照固定的大小（blocksize）切成若干块后分布式存储在若干台datanode上
5.每一个文件块可以有多个副本，并存放在不同的datanode上
副本不会放在同一个机器上，因为副本就是防止宕机，
6.Datanode会定期向Namenode汇报自身所保存的文件block信息，而namenode则会负责保持文件的副本数量
因为datanode如果宕机的话，name该机器上的对应的副本数据将会消失，这样需要将其在其他机器上进行恢复，恢复的话，就需要上面就需要数据和未宕机时的数据尽量保持一致，所以需要依赖于datanode定期汇报，不然差距的数据会很大
7.HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是通过向namenode申请来进行



## Hdfs写操作

 详细步骤解析

1、根namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在

不存在则会返回path not exist异常

2、namenode返回是否可以上传

3、client请求第一个 block（0-128m）该传输到哪些datanode服务器上

返回该block存放的位置，及其副本的信息存放的位置

4、namenode返回3个datanode服务器ABC

副本选择策略

考虑空间和距离的因素，网络跳转的跳数，比如说机架的位置，

第一台是看谁比较近（机架），因为传输比较快，副本则是是看谁比较远，防止机架出问题（如断电），干扰性更小

而集群全线崩塌

5、client请求3台dn中的一台A上传数据（本质上是一个RPC调用，建立pipeline），A收到请求会继续调用B，然后B调用C，将真个pipeline建立完成，逐级返回客户端

这样是防止整个流程变慢，同时创建通道，先建立通道pipeline,通道

6、client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存bytebuf），以packet为单位，A收到一个packet就会传给B，B传给C；A每传一个packet会放入一个应答队列等待应答

 

因为等一个block写满之后再传送，速度会很慢，所以是接收一个packet就会写入到管道流pipeline中。

只要上传一个成功，则客户端视为上传成功，因为如果没上传成功，namenode会进行异步的复制副本的信息

7、当一个block传输完成之后，client再次请求**namenode**上传第二个block的服务器。

注：写的过程中，namenode记录下来了文件路径，文件有几个block也记录下来了，每个block分配到哪些机器上也记录下到了，及其每个block的副本信息，副本在那几个机器上。

校验的时候不是一个packet（一批chunk，共64k）校验，而是以一个chunk来校验，一个chunk是512byte（字节）

# Hdfs读操作

1、跟namenode通信查询元数据，找到文件块所在的datanode服务器

2、挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流

3、datanode开始发送数据（从磁盘里面读取数据放入流，以packet为单位来做校验）

4、客户端以packet为单位接收，现在本地缓存，然后写入目标文件
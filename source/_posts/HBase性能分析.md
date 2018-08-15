---
title: HBase原理性能分析
date: 2018年08月06日 22时15分52秒
tags: [HBase,原理]
categories: 大数据
toc: true
---

[TOC]





# HBase介绍

HBase表很大：一个表可以有数十亿行，上百万列；

HBase的表将会分成很多个分区，每个分区部分会存在不同的机器上 
分区是为了便于查询，放在不同机器上，io也增大，假如一个机器的io的是100m，两个就为200m，读取速度就变快了==>**多台机器的io能得到充分利用**



HBase表无模式：每行都有一个可排序的主键好任意多的列，列可以根据需要动态的增加，同一张表中不同的行可以有不同的列；

面向列：列独立检索；

稀疏：空列并不占用存储空间，表可以设计的非常稀疏；

数据类型单一：HBase中的数据都是字符串，没有类型

<!-- more -->

HBase采用类LSM的架构体系，数据写入并没有直接写入数据文件，而是会先写入缓存（Memstore），在满足一定条件下缓存数据再会异步刷新到硬盘。为了防止数据写入缓存之后不会因为RegionServer进程发生异常导致数据丢失，在写入缓存之前会首先将数据顺序写入HLog中。如果不幸一旦发生RegionServer宕机或者其他异常，这种设计可以从HLog中进行日志回放进行数据补救，保证数据不丢失。HBase故障恢复的最大看点就在于如何通过HLog回放补救丢失数据。



# HBase结构

HBase进行存储的服务器

HRegion是HBase当中的一个类，**一个表分区的类，按照行分区** 
一个HRegion只会在一个HBase上，一个HBase上可以有多个HRegion 
HBase表每个分区（按照行来分区）的数据被封装到一个类HRegion内 
如：HBase存在user表，role表，共4个regionServer，HRegion1存储管理user表的一部分，HRegion2存储管理user表的一部分，HRegion3存储管理role表的一部分，HRegion4存储管理role表的一部分

## HRegionserver

管理用户对Table的增、删、改、查操作；
记录region在哪台Hregion server上
在Region Split后，负责新Region的分配；
新机器加入时，管理HRegion Server的负载均衡，调整Region分布
在HRegion Server宕机后，负责失效HRegion Server 上的Regions迁移。



## HBase

HRegion Server主要负责响应用户I/O请求，向HDFS文件系统中读写数据，是HBase中最核心的模块。

HRegion Server管理了很多table的分区，也就是region。

## HRegion构成

HRegion类中有**HLog，store**成员，分别代表硬盘和内存 

### Store

每个Region包含着多个Store对象，一个列簇对应一个store 。每个Store包含一个MemStore和若干StoreFile，StoreFile包含一个或多个HFile，StoreFile是对HFile的一种封装。MemStore存放在内存中，StoreFile存储在HDFS上。

### HLog

HLog最终是放在hdfs上。

当我们客户端上传一个表名，一个列簇，一个值，这条命令的值会原封不动的将其写入到HLog里面， 这个是一个appendLog,只可以从底部追加，不允许修改，写到HLog之后，再将数据写入到内存（memstore）当中，HLog里面是存储的操作信息的数据 写在HLog中是因为，防止在写入到内存中的时候，宕机

## Region的划分

Region按大小分割的，随着数据增多，Region不断增大，当增大到一个阀值（默认256m）的时候，Region就会分成两个新的Region



## HRegion的存储

### ROOT表和META表

HBase的所有Region元数据被存储在.META.表中，随着Region的增多，.META.表中的数据也会增大，并分裂成多个新的Region。为了定位.META.表中各个Region的位置，把.META.表中所有Region的元数据保存在-ROOT-表中，最后由Zookeeper记录-ROOT-表的位置信息。所有客户端访问用户数据前，需要首先访问Zookeeper获得-ROOT-的位置，然后访问-ROOT-表获得.META.表的位置，最后根据.META.表中的信息确定用户数据存放的位置，如下图所示。
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fuag5ugi8yj30y60ds0tw.jpg)

-ROOT-表永远不会被分割，它只有一个Region，这样可以保证最多只需要三次跳转就可以定位任意一个Region。为了加快访问速度，.META.表的所有Region全部保存在内存中。客户端会将查询过的位置信息缓存起来，且缓存不会主动失效。如果客户端根据缓存信息还访问不到数据，则询问相关.META.表的Region服务器，试图获取数据的位置，如果还是失败，则询问-ROOT-表相关的.META.表在哪里。最后，如果前面的信息全部失效，则通过ZooKeeper重新定位Region的信息。所以如果客户端上的缓存全部是失效，则需要进行6次网络来回，才能定位到正确的Region。



root表,mate表都不会很大 
因为root表，只是记录位置，本身就不会太大， 
**meta表，和root表在过程中都会被加载到内存中** 
经过3次来回，总共六次，会得到数据表的位置 
1，client向zookeeper获取root表的位置 2，zookeeper返回root表地址信息 
3，client读取root表，获得table1的meta表的地址，4 root表所在机器返回meta表地址 
5，client向mete表读取table地址，6 client向table插入数据 
读取和写入都会经历上面的过程



# HBase写数据流程

client向HRegionserver发送写请求。 
HRegionserver将**操作信息**数据写到hlog（write ahead log）。为了数据的持久化和恢复。 HLog记录的是操作数据 

HRegionserver将**实际数据**写到内存（memstore） 
反馈client写成功。



## 数据flush

当memstore数据达到阈值64（新版本默认是128M），将内存集合中的数据刷到硬盘，将内存中的数据删除，同时删除Hlog中的历史数据。 
并将数据存储到hdfs中。以**数据块**的形式存储。 
在hlog中做标记点。

### flush的说明

当内存文件memstore文件达到64m的时候，会将数据合并刷新写入到StroeFile文件里面，再将数据写入到HFile里面，HFile文件是一个hdfs文件，序列化到hdfs里面，再通过hdfs的api写入到hdfs集群里面 
提交到hdfs集群后，HLog，memstore的数据将会被清除



## 数据合并

1、当（hdfs中）数据块达到4块，hmaster将数据块**加载到本地**(HRegionserver)，进行合并 
注：这个数据块单块没有大小限制 
2、当合并的数据超过256M，进行拆分，将拆分后的region分配给不同的hregionserver管理 
注：如果不大于256M，将数据原封不动写回hdfs 
3、当hregionsever**宕机**后，将该hregionserver上的hlog拆分（按表拆分)，然后分配给不同的hregionserver加载，修改.META. 
注：由Hmaster来更改.META.文件，不会对HRegionserver的性能造成影响 
4、注意：hlog会同步到hdfs 
注：合并的数据是对相同rowkey的分组合并，如user表中，对id='1'的操作内容合并，对id='2'的合并



### 合并的好处

**合并操作是由hmaster来工作** 
合并操作是针对一个表来说，user表合并user表，role表合并role表 
1、清理了垃圾数据 
2、将大的数据块拆分后，给多个机器管理，优化读取操作等速率

# HBase的读流程

通过zookeeper和-ROOT- .META.表定位HBase。
数据从内存和硬盘合并后返回给client
数据块会缓存





# Client

HBase Client使用HBase的RPC机制与HRegionserver和RegionServer进行通信
管理类操作：Client与HRegionserver进行RPC；
数据读写类操作：Client与HBase进行RPC。







# 读取数据与写入数据流程的不同

除了在表中读取数据，还要在内存磁盘上去搜寻还未存到hdfs里面的数据。 
所以数据块太大的应该拆分，可以加快查询速度 
经常查询的数据块还应该放在内存中（HRegionserver的内存中）



# HBase的出现

hdfs是分布式文件系统，只能保存整个文件，如果一行一行的保存数据，namenode的压力会很大 
假如有一个文件下有100w小个文件，每个文件都是1k，在datanode中不会占用128m的分块大，但是每个文件元数据所占的大小是一样的，这样的话，namenode空间占满时，datanode中的数据实际上很少。 
HBase就会很好的解决这个问题，一个文件一个文件写的时候，是先写入到HBase中，先写入到HBase集群中（HBase也分为主从，主为HMaster，从为HRegionserver） 
，HRegionserver的内存中，当数据量达到128m的时候，将数据写入到hdfs中，这样128m数据的元数据只有一条

# HBase细节

HBase实际上是一个缓存层，存储的数据量很少，存的一部分缓存的数据，HBase需要zookeeper来定位HBase查找数据的偏移量

# HBase 主从之间的关系

hmaster只是一个管理者，而且只管理，当HBase集群挂掉之后，数据偏移信息和表的信息，而不管理数据信息，所以有一种极端情况，当HBase集群启动之后，表创建完成，正常运行之后，将hmaster关闭也不会影响整个集群的运行。不像namenode挂了之后不能响应了

# note：

HBase写快读慢，读慢是相对于写来说的，但是跟mysql相比，也不是一个量级的 




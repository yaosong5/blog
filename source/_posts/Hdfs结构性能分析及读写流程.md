---
title: Hdfs结构性能分析及读写流程
date: 2018年08月06日 22时15分52秒
tags: [HDFS,原理,Hadoop]
categories: 大数据
toc: true
---

[TOC]

# hdfs的设计思想
分而治之：将大文件、大批量文件，分布式存放在大量服务器上，以便于采取分而治之的方式对海量数据进行运算分析
首先，它是一个文件系统，用于存储文件，通过统一的命名空间——目录树来定位文件
其次，它是分布式的，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色；

<!-- more -->

# hdfs功能和特点


hdfs的重要特性如下：

* HDFS中的文件在物理上是分块存储（block），块的大小可以通过配置参数( dfs.blocksize)来规定，默认大小在hadoop2.x版本中是128M，老版本中是64M

* HDFS文件系统会给客户端提供一个统一的抽象目录树，客户端通过路径来访问文件，形如：hdfs://namenode:port/dir-a/dir-b/dir-c/file.data

* 目录结构及文件分块信息(元数据)的管理由namenode节点承担
  ——namenode是HDFS集群主节点，负责维护整个hdfs文件系统的目录树，以及每一个路径（文件）所对应的block块信息（block的id，及所在的datanode服务器）


* 文件的各个block的存储管理由datanode节点承担---- datanode是HDFS集群从节点，每一个block都可以在多个datanode上存储多个副本（副本数量也可以通过参数设置dfs.replication）

HDFS是设计成适应一次写入，多次读出的场景，且不支持文件的修改


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



# Hdfs写操作
![](http://pebgsxjpj.bkt.clouddn.com/15361391927714.jpg)


 详细步骤解析

1、根namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在

不存在则会返回path not exist异常

2、namenode返回是否可以上传

3、client请求第一个 block（0-128m）该传输到哪些datanode服务器上

返回该block存放的位置，及其副本的信息存放的位置

4、namenode返回3个datanode服务器ABC

副本选择策略（如果设置为被分数为2的话）

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
![](http://pebgsxjpj.bkt.clouddn.com/15361460237204.jpg)

客户端将要读取的文件路径发送给namenode，namenode获取文件的元信息（主要是block的存放位置信息）返回给客户端，客户端根据返回的信息找到相应datanode逐个获取文件的block并在客户端本地进行数据追加合并从而获得整个文件

1、跟namenode通信查询元数据，找到文件块所在的datanode服务器

2、挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流

3、datanode开始发送数据（从磁盘里面读取数据放入流，以packet为单位来做校验）

4、客户端以packet为单位接收，现在本地缓存，然后写入目标文件


对此，我们了解了hdfs的读写流程，那么我们再来看看hdfs元数据的管理


## 小贴士



**hdfs中datanode的初始化**

hdfs会在配置文件中配置一个namenode的工作目录元数据 

查看目录结构 tree $DATANODE/ 
![](http://pebgsxjpj.bkt.clouddn.com/15361570887042.jpg)



datanode的工作目录是在datanode启动后初始化的 
而hadoop namenode format 只会初始name的工作目录，和datanode没有关系

**如何把一个hdfs的一个节点加入到另一个集群**
因为在原来的目录中会有原来集群的信息如：ClusterID
![](http://pebgsxjpj.bkt.clouddn.com/15361571791381.jpg)

必须要将hdfs datanode的工作目录删除，不然持有上一个集群的datanode的工作目录，会认为是一个误操作，为了防止丢失数据，不会让其连接上



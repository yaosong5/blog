---
title: Hadoop零碎知识点
date: 2018年08月06日 22时15分52秒
tags: [HDFS,原理,Hadoop]
categories: 大数据
toc: true
---

[TOC]

## 查看元数据信息
可以通过hdfs的一个工具来查看edits中的信息

```bash
bin/hdfs oev -i edits -o edits.xml
bin/hdfs oiv -i fsimage_0000000000000000087 -p XML -o fsimage.xml
```
<!-- more -->

# 查看目录树

```bash
hdfs会在配置文件中配置一个datanode的工作目录元数据 
查看目录结构 tree hddata/ 
```



# 安全模式

![](https://ws1.sinaimg.cn/large/006tNbRwly1fubqgn0culj31eo0nm401.jpg)

这是因为在分布式文件系统启动的时候，开始的时候会有安全模式，当分布式文件系统处于安全模式的情况下，文件系统中的内容不允许修改也不允许删除，直到安全模式结束。安全模式主要是为了系统启动的时候检查各个DataNode上数据块的有效性，同时根据策略必要的复制或者删除部分数据块。运行期通过命令也可以进入安全模式。在实践过程中，系统启动的时候去修改和删除文件也会有安全模式不允许修改的出错提示，只需要等待一会儿即可。

# 安全模式的出现

为甚么集群启动的时候要进入安全模式，因为不进入安全模式，nameNode不知道数据块存放在哪儿

为什么一开始，namenode不记录下这些数据的地址呢？因为数据位置信息会发生变化

这些数据记录在datanode的block块下的元数据下。fodaration模式是记录在juoinrnode下的

juoinrnode也是一个硬盘版的zookeeper

在hdfs启动的时候，处于安全模式，datanode向namenode汇报自己的ip和持有的block块信息，这样，安全模式结束后，文件块和datanode的ip关联上，存放数据的位置也就可以找到





可以通过以下命令来手动离开安全模式：

```
bin/hadoop dfsadmin -safemode leave  
```

用户可以通过dfsadmin -safemode value 来操作安全模式，参数value的说明如下： 
enter - 进入安全模式 
leave - 强制NameNode离开安全模式 
get - 返回安全模式是否开启的信息 
wait - 等待，一直到安全模式结束。

# 还有另外一种情况

当namenode发现集群中的block丢失数量达到一个阀值时，namenode就进入安全模式状态，不再接受客户端的数据更新请求

在正常情况下，namenode也有可能进入安全模式： 
集群启动时（namenode启动时）必定会进入安全模式，然后过一段时间会自动退出安全模式（原因是datanode汇报的过程有一段持续时间）

也确实有异常情况下导致的安全模式 
原因：block确实有缺失 
措施：可以手动让namenode退出安全模式，bin/hdfs dfsadmin -safemode leave 
或者：调整safemode门限值： dfs.safemode.threshold.pct=0.999f





# Hadoop的调度器

FIFO: 默认，先进先出的原则

Capacity： 计算能力调度器，选择占用最小、优先级高的执行

Fair: 公平调度，所有的job具有相同的资源

（可以在core-site.xml中配置，spark是有两种调度器，fifo 公平调度）
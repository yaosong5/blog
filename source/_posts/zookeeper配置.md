---
title:  Zookeeper的配置容器的搭建
date: 2018年08月06日 22时15分52秒
tags:  [zk,Docker]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---

[TOC]
![](http://img.gangtieguo.cn/006tNbRwgy1fu554uqh0pj31660fwta8.jpg)

在usr目录下下载zk包，并且解压到/usr/目录，改名为zk，所以$ZK_HOME为/usr/zk

# 创建目录

```
mkdir -p /usr/zk/data
mkdir -p /usr/zk/logs
touch /usr/zk/data/myid
```

<!--more -->

# 更改配置文件

`vim  $ZK_HOME/conf/zoo.cfg`

```
dataDir=/usr/zk/data
dataLogDir=/usr/zk/logs
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

zookeeper需要全部左右节点都启动才会选举leader，follower

所有节点启动的脚本

```bash
# !/bin/bash
echo 1 > $ZK_HOME/data/myid
$ZK_HOME/bin/zkServer.sh start
ssh root@zk2 "echo 2 > /usr/zk/data/myid"
ssh root@zk2 "/usr/zk/bin/zkServer.sh start"
ssh root@zk3 "echo 3 > /usr/zk/data/myid"
ssh root@zk3 "/usr/zk/bin/zkServer.sh start"
```





# zk问题

1、由于 zk 运行一段时间后，会产生大量的日志文件，把磁盘空间占满，导致整个机器进程都不能活动了，所以需要定期清理这些日志文件，方法如下：

1）、写一个脚本文件 cleanup.sh 内容如下：

```
 java -cp zookeeper.jar:lib/slf4j-api-1.6.1.jar:lib/slf4j-log4j12-1.6.1.jar:lib/log4j-1.2.15.jar:conf org.apache.zookeeper.server.PurgeTxnLog <dataDir> <snapDir> -n <count>
 其中：

　　dataDir：即上面配置的 dataDir 的目录
　　
　　snapDir：即上面配置的 dataLogDir 的目录

　　count：保留前几个日志文件，默认为 3


```

2）、通过 crontab 写定时任务，来完成定时清理日志的需求

```
crontab -e 0 0 * *  /opt/zookeeper-3.4.10/bin/cleanup.sh
HBase Master 高可用（HA）（http://www.cnblogs.com/captainlucky/p/4710642.html）
HMaster 没有单点问题，HBase 中可以启动多个 HMaster，通过 Zookeeper 的 Master Election 机制保证总有一个 Master 运行。
```

所以这里要配置 HBase 高可用的话，只需要启动两个 HMaster，让 Zookeeper 自己去选择一个 Master Acitve。
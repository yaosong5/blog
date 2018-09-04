---
title:  FLINK容器的搭建
date: 2018年08月06日 22时15分52秒
tags:  [Docker,FLINK]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---

[TOC]





# 来源容器  flk

新建容器，为减少工作量，引用的是有ssh服务的Docker镜像**kinogmt/centos-ssh:6.7**，生成容器os为基准。

```
docker run -itd  --name flk --hostname flk kinogmt/centos-ssh:6.7 &> /dev/null
```

> 注意必须要以-d方式启动，不然sshd服务不会启动，这算是一个小bug

<!--more-->

在容器中下载需要的elk的源包。做解压就不赘述，很多案例教程。

> 我是采用的下载到宿主机，解压后，用 "docker cp 解压包目录  os:/usr/loca/"来传到容器内，比在容器内下载速度更快
>
> ## 复制源包
>
> ```
> docker cp /Users/yaosong/Yao/hadoop  flk:/usr/
> docker cp /Users/yaosong/Yao/flink  flk:/usr/
> ```





# 配置home


	export JAVA_HOME=/usr/java/jdk1.8.0_144/
	export PATH=$JAVA_HOME/bin:$PATH
	export FLINK_HOME=/usr/flink
	export HADOOP_HOME=/usr/hadoop
	export HADOOP_CONFIG_HOME=$HADOOP_HOME/etc/hadoop
	export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
	export PATH=$PATH:$HADOOP_HOME/bin
	export PATH=$PATH:$HADOOP_HOME/sbin
	export PATH=$PATH:$FLINK_HOME/bin

**配置hadoop**  参考：

[]: http://gangtieguo.cn/2018/07/20/Docker%E4%B8%ADhadoop%20spark%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/	"Docker 中 hadoop，spark 镜像搭建"



**flink.yml**

```
 high-availability: zookeeper

# The path where metadata for master recovery is persisted. While ZooKeeper stores
# the small ground truth for checkpoint and leader election, this location stores
# the larger objects, like persisted dataflow graphs.
# 
# Must be a durable file system that is accessible from all nodes
# (like HDFS, S3, Ceph, nfs, ...) 
#
 high-availability.storageDir: hdfs:///flink/ha/

# The list of ZooKeeper quorum peers that coordinate the high-availability
# setup. This must be a list of the form:
# "host1:clientPort,host2:clientPort,..." (default clientPort: 2181)
#
high-availability.zookeeper.quorum: zk1:2181,zk2:2181,zk3:2181
```

**Master**

```
master:8081
```

**slave**

```
slave01
slave02
```

**zoo.cfg**

```
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

**需要在yarn-site.xml中配置**

```xml
<property>
 	<name>yarn.resourcemanager.am.max-attempts</name>
	<value>4</value>
</property>
<property>
	<name>yarn.nodemanager.resource.cpu-vcores</name>
	<value>8</value>
</property>
```

# 保存镜像

```bash
docker commit -m "bigdata:flink,hadoop"  --author="yaosong"  flk  yao/flinkonyarn:1.0
```

#   获得flk 容器

```bash
 docker run -itd --net=br --name flk1 --hostname flk1 yao/flinkonyarn:1.0 &> /dev/null
 docker run -itd --net=br --name flk2 --hostname flk2 yao/flinkonyarn:1.0 &> /dev/null
 docker run -itd --net=br --name flk3 --hostname flk3 yao/flinkonyarn:1.0 &> /dev/null
```

# 停止/删除flk 容器

```bash
docker stop flk1
docker stop flk2
docker stop flk3
docker rm flk1
docker rm flk2
docker rm flk3
```

# 官方：wordcount

## flink测试命令

由于在本地搭建，机器配置有限，故设置不同参数命令来运行官方wordcount

```bash
flink run -m yarn-cluster $FLINK_HOME/examples/batch/WordCount.jar

flink run -m yarn-cluster -ynd 2 $FLINK_HOME/examples/batch/WordCount.jar

flink run -m yarn-cluster -yn 4  $FLINK_HOME/examples/batch/WordCount.jar

flink run -m yarn-cluster -yn 6  $FLINK_HOME/examples/batch/WordCount.jar

flink run -m yarn-cluster -yn 8  $FLINK_HOME/examples/batch/WordCount.jar

flink run -m yarn-cluster -yn 10  $FLINK_HOME/examples/batch/WordCount.jar
```








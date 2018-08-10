---
title:  HBase容器的搭建
date: 2018年08月06日 22时15分52秒
tags:  [HBase,Docker]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---


[TOC]

# 创建hbasezk镜像
##   拷贝源码


```bash
docker cp /Users/yaosong/Yao/hbase 8019587d559b:/usr/
docker cp /Users/yaosong/Yao/zk   8019587d559b:/usr/
docker cp /Users/yaosong/Yao/hbasezkStart.sh 8019587d559b:/usr/
docker cp /Users/yaosong/Yao/hbase-start.sh 8019587d559b:/usr/
docker cp /Users/yaosong/Yao/zk-start.sh 8019587d559b:/usr/
```

参考https://www.cnblogs.com/netbloomy/p/6677883.html

## 解压

`tar -zxvf hbase-1.3.0-bin.tar.gz`
进入 hbase 的配置目录，在 hbase-env.sh 文件里面加入 java 环境变量.

 即：

```bash
vim  hbase-env.sh
export JAVA_HOME=JAVA_HOME=/usr/java/jdk
```

关闭 HBase 自带的 Zookeeper, 使用 Zookeeper 集群：

```
vim  hbase-env.sh
export  HBASE_MANAGES_ZK=false
```

hbase-site.xml

```xml
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://master:9000/hbase</value>
    </property>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>hbasezk1,hbasezk2,hbasezk3</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/usr/hbase/tmp/zk/data</value>
    </property>
    <!-- webui的配置 -->
    <property>
        <name>hbase.master.info.port</name>
        <value>60010</value>
    </property>
    <!-- webui新增的配置 -->
</configuration>
```



创建zk的datadir目录

`mkdir -p /usr/hbase/tmp/zk/data`


编辑配置目录下面的文件 regionservers. 命令：

	vim   $HBASE_HOME/config/regionservers

	加入如下内容：
	hbasezk1
	hbasezk2
	hbasezk3


	把 Hbase 复制到其他机器scp

	开启 hbase 服务。命令如下： 哪台上运行哪台就为hmaster

 	$HBASE_HOME/bin/start-hbase.sh

	在 hbasezk1,2,3 中的任意一台机器使用 $HBASE_HOME/bin/hbase shell

	进入 hbase 自带的 shell 环境，然后使用命令 version 等，进行查看 hbase 信息及建立表等操作。
要配置 HBase 高可用的话，只需要启动两个 HMaster，让 Zookeeper 自己去选择一个 Master Acitve。

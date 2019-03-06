---
title: yaosong5/bigdata:2.0-Hadoop&Spark&hive&zk&hue组合容器的搭建
date: 2018年08月06日 22时15分52秒
tags:  [Docker,Spark,Hadoop]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---

[TOC]

**配置centos集群 hadoop spark组件**
启动容器
各组件版本对应
hbase1.2  hive 版本 2.0.0  hbase1.x ZooKeeper 3.4.x is required as of HBase 1.0.0



新建容器，为减少工作量，引用的是有ssh服务的Docker镜像**kinogmt/centos-ssh:6.7**，生成容器os为基准。

```
docker run -itd  --name bigdata --hostname bigdata kinogmt/centos-ssh:6.7 &> /dev/null
```

> 注意必须要以-d方式启动，不然sshd服务不会启动，这算是一个小bug

<!--more-->

在容器中下载需要的elk的源包，做解压就不赘述，很多案例教程。

 我是采用的下载到宿主机，解压后，用 "docker cp 解压包目录  os:/usr/loca/"来传到容器内，比在容器内下载速度更快

 # 拷贝文件到容器

 

 命令格式`docker cp 本地文件路径   容器id或者容器名称:`
 将所有组件下载解压并拷贝到容器

 ```bash
 docker cp /Users/yaosong/Downloads/hadoop-2.8.0.tar.gz   bigdata:/
 docker cp /Users/yaosong/Downloads/spark-2.2.0-bin-without-hadoop.tgz   bigdata:/
 docker cp /Users/yaosong/Downloads/jdk-8u144-linux-x64.rpm   bigdata:/
 docker cp /Users/yaosong/Downloads/spark-2.1.0-bin-hadoop2.6.tgz   bigdata:/
 docker cp  /Users/yaosong/Yao/spark源包/hive bigdata:/usr
 docker cp /Users/yaosong/Downloads/jdk-8u144-linux-x64.rpm  bigdata:/
 docker cp /Users/yaosong/Downloads/hadoop-2.8.0.tar.gz  bigdata:/
 docker cp /Users/yaosong/Downloads/spark-2.2.0-bin-without-hadoop.tgz  bigdata:/
 docker cp /Users/yaosong/Downloads/spark-2.1.0-bin-hadoop2.6.tgz  bigdata:/
 docker cp /Users/yaosong/Yao/spark源包/hbase  bigdata:/usr
 docker cp /Users/yaosong/Yao/spark源包/zk   bigdata:/usr
 docker cp  /Users/yaosong/Yao/ant    bigdata:/usr
 docker cp  /Users/yaosong/Yao/maven  bigdata:/usr
 docker cp  /Users/yaosong/Yao/hue4  bigdata:/usr
 创建home

 vim /etc/profile

 mac: vim ~/.bashrc

 添加以下内容

     export JAVA_HOME=/usr/java/jdk
     export PATH=$JAVA_HOME:$PATH
     export SCALA_HOME=/usr/scala-2.12.3/
     export HADOOP_HOME=/usr/hadoop
     export HADOOP_CONFIG_HOME=$HADOOP_HOME/etc/hadoop
     export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
     export PATH=$PATH:$HADOOP_HOME/bin
     export PATH=$PATH:$HADOOP_HOME/sbin
     export SPARK_DIST_CLASSPATH=$(hadoop classpath)
     SPARK_MASTER_IP=master
     SPARK_LOCAL_DIRS=/usr/spark
     SPARK_DRIVER_MEMORY=1G
     export SPARK_HOME=/usr/spark
     export PATH=$SPARK_HOME/bin:$PATH
     export PATH=$SPARK_HOME/sbin:$PATH
     
     MAVEN_HOME=/usr/maven
     export MAVEN_HOME
     export PATH=${PATH}:${MAVEN_HOME}/bin
     ANT_HOME=/usr/ant
     PATH=$JAVA_HOME/bin:$ANT_HOME/bin:$PATH
     export ANT_HOME PATH
     HUE_HOME=/usr/hue4
     export ZK_HOME=/usr/zk
     export HBASE_HOME=/usr/hbase
     export PATH=$HBASE_HOME/bin:$PATH
     export PATH=$ZK_HOME/bin:$PATH

 ```



# 安装

## 创建 hadoop 集群所需目录：

在以下配置文件中会有以下目录

```bash
cd $HADOOP_HOME;
mkdir tmp
mkdir namenode
mkdir datanode
cd $HADOOP_CONFIG_HOME/
```
## 更改配置文件

`cd $HADOOP_CONFIG_HOME/` or `cd $HADOOP_HOME/etc/hadoop`
#### hdfs slaves

```bash
slave01
slave02
```

#### core-site.xml：

```xml
<property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/hadoop/tmp</value>
        <description>A base for other temporary directories.</description>
</property>
<property>
	<name>fs.default.name</name>
	  <value>hdfs://master:9000</value>
	  <final>true</final>
	  <description>The name of the default file system.
	  A URI whose scheme and authority determine the
	  FileSystem implementation.
	  </description>
  </property>
  <!--hive的配置，参考https://blog.csdn.net/lblblblblzdx/article/details/79760959-->
  <property>
  	<name>hive.server2.authentication</name>
  	<value>NONE</value>
  </property>
  <!--hive的配置hadoop代理用户  root用户提交的任务可以在任意机器上以任意组的所有用户的身份执行。-->
  <property>
 	<name>hadoop.proxyuser.root.hosts</name>
  	<value>*</value>
  </property>
  <property>
	<name>hadoop.proxyuser.root.groups</name>
	<value>*</value>
  </property>
	 <!--HUE 增加一个值开启 hdfs 的 web 交互-->
	<property>
	      <name>dfs.webhdfs.enabled</name>
	      <value>true</value>
	</property>
 <!--HUE 增加一个值开启 hdfs 的 web 交互-->
```

#### hdfs-site.xml：
```xml
<property>
    <name>dfs.replication</name>
    <value>2</value>
    <final>true</final>
    <description>Default block replication.
    The actual number of replications can be specified when the file is created.
    The default is used if replication is not specified in create time.
    </description>
</property>

<property>
    <name>dfs.namenode.name.dir</name>
    <value>/usr/hadoop/namenode</value>
    <final>true</final>
</property>

<property>
    <name>dfs.datanode.data.dir</name>
    <value>/usr/hadoop/datanode</value>
    <final>true</final>
</property>
<!--《为了让 hue 能够访问 hdfs，需要在 hdfs-site.xml 里面配置一些内容-->
<property>
    <name>hadoop.proxyuser.hue.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.hue.groups</name>
    <value>*</value>
</property>
<!--《为了让 hue 能够访问 hdfs，需要在 hdfs-site.xml 里面配置一些内容-->
```

#### mapred-site.xml：
```xml
<property>
    <name>mapred.job.tracker</name>
    <value>master:9001</value>
    <description>The host and port that the MapReduce job tracker runs
    at.  If "local", then jobs are run in-process as a single map
    and reduce task.
    </description>
</property>
```
####  yarn-site.xml：
```xml
<property>
	<name>yarn.nodemanager.pmem-check-enabled</name>
	<value>false</value>
</property>
<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
    <description>Whether virtual memory limits will be enforced for containers</description>
</property>
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>256</value>
</property>
<property>
	<name>yarn.resourcemanager.address</name>
	<value>master:8032</value>
</property>
<property>
	<name>yarn.resourcemanager.scheduler.address</name>
	<value>master:8030</value>
</property>
<property>
	<name>yarn.resourcemanager.resource-tracker.address</name>
	<value>master:8031</value>
</property>
```



如果是hadoop3以上版本，需要在**start-dfs.sh start-yarn.sh**中开头空白处分别配置一下内容

```bash
vim $HADOOP_HOME/sbin/start-dfs.sh

HDFS_DATANODE_USER=root
HADOOP_SECURE_DN_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root

vim $HADOOP_HOME/sbin/start-yarn.sh

YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=root
YARN_NODEMANAGER_USER=yarn
YARN_PROXYSERVER_USER=root
```

## 格式化namenode

```bash
$HADOOP_HOME/bin/hadoop namenode -format
```


# 启动集群

`$HADOOP_HOME/sbin/start-all.sh`

测试

```bash
yarn 8088端口   http://yourip:8088
```

```bash
hdfs 50070端口 hdfs3.0为9870   http://yourip:50070
```

## spark只需要在slaves中添加

```bash
slave01
slave02
```
**sparkUI端口8080**

# 测试spark集群

启动spark 

```bash
$HADOOP_HOME/bin/start-all.sh
```

## 官网命令

```bash
$SPARK_HOME/bin/spark-submit --class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
--driver-memory 512m \
--executor-memory 512m \
--executor-cores 1 \
$SPARK_HOME/examples/jars/spark-examples*.jar \
10
```



## 执行spark on yarn命令行模式

```bash
spark-shell --master yarn --deploy-mode client --driver-memory 1g --executor-memory 1g --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 512m --executor-memory 512m --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 475m --executor-memory 475m --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 350m --executor-memory 350m --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 650m --executor-memory 650m --executor-cores 1
```



# 创建镜像

```bash
docker commit -m "bigdata基础组件镜像"  bigdata yaosong5/bigdata:2.0
```

2.0镜像对比1.0：1.0只有hadoop-待确定

# 创建容器

```bash
docker run -itd  --net=br  --name master --hostname master yaosong5/bigdata:2.0 &> /dev/null
docker run -itd  --net=br  --name slave01 --hostname slave01 yaosong5/bigdata:2.0 &> /dev/null
docker run -itd  --net=br  --name slave02 --hostname slave02 yaosong5/bigdata:2.0 &> /dev/null
```

# 停止and 删除容器

```bash
docker stop master
docker stop slave01
docker stop slave02
docker rm master
docker rm slave01
docker rm slave02
```


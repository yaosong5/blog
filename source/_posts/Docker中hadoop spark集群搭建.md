---
title:  Docker中hadoop，spark镜像搭建
date: 2018年07月20日 02时54分02秒
tags:  [Docker,Hadoop,Spark]
categories: 部署安装
toc: true
---
![enter description here](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/7/20/1532052420904.jpg)

[toc]



**配置centos集群 hadoop spark组件**
启动容器
各组件版本对应
hbase1.2  hive 版本 2.0.0  hbase1.x ZooKeeper 3.4.x is required as of HBase 1.0.0


<!-- more -->
# 拷贝文件到容器
命令格式`docker cp 本地文件路径   容器id或者容器名称:`
将所有组件下载解压并拷贝到容器
```bash
docker cp /Users/yaosong/Downloads/hadoop-2.8.0.tar.gz   yaobigdata:/
docker cp /Users/yaosong/Downloads/spark-2.2.0-bin-without-hadoop.tgz   yaobigdata:/
docker cp /Users/yaosong/Downloads/jdk-8u144-linux-x64.rpm   yaobigdata:/
docker cp /Users/yaosong/Downloads/spark-2.1.0-bin-hadoop2.6.tgz   yaobigdata:/
docker cp  /Users/yaosong/Yao/spark源包/hive yaobigdata:/usr
docker cp /Users/yaosong/Downloads/jdk-8u144-linux-x64.rpm  yaobigdata:/
docker cp /Users/yaosong/Downloads/hadoop-2.8.0.tar.gz  yaobigdata:/
docker cp /Users/yaosong/Downloads/spark-2.2.0-bin-without-hadoop.tgz  yaobigdata:/
docker cp /Users/yaosong/Downloads/spark-2.1.0-bin-hadoop2.6.tgz  yaobigdata:/
docker cp /Users/yaosong/Yao/spark源包/hbase  yaobigdata:/usr
docker cp /Users/yaosong/Yao/spark源包/zk   yaobigdata:/usr
docker cp  /Users/yaosong/Yao/ant    yaobigdata:/usr
docker cp  /Users/yaosong/Yao/maven  yaobigdata:/usr
docker cp  /Users/yaosong/Yao/hue4  yaobigdata:/usr
```

# 创建home

`vim /etc/profile`
mac: vim ~/.bashrc
添加以下内容
```bash
export JAVA_HOME=/usr/java/jdk1.8.0_144/
export PATH=$JAVA_HOME:$PATH
export SCALA_HOME=/usr/scala-2.12.3/
export HADOOP_HOME=/usr/hadoop
export HADOOP_CONFIG_HOME=$HADOOP_HOME/etc/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
export SPARK_DIST_CLASSPATH=$(hadoop classpath)
SPARK_MASTER_IP=master
SPARK_LOCAL_DIRS=/usr/spark-2.2.0-bin-without-hadoop
SPARK_DRIVER_MEMORY=1G
export SPARK_HOME=/usr/spark-2.2.0-bin-without-hadoop
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

# 创建镜像

docker commit -m "bigdata + hue + zk + kafka"  mm   yaosong5/bigdata:2.0

# 创建容器

```
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



# 安装
## 准备

### 创建 hadoop 集群所需目录：
```bash
	cd $HADOOP_HOME;
	mkdir tmp
	mkdir namenode
	mkdir datanode
	cd $HADOOP_CONFIG_HOME/
```
### 更改配置文件
cd $HADOOP_CONFIG_HOME/
#### hdfs slaves

	slave01
	slave02

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
            FileSystem implementation. The uri's scheme
            determines the config property (fs.SCHEME.impl)
            naming the FileSystem implementation class. The
            uri's authority is used to determine the host,
            port, etc. for a filesystem.
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
      <value>256mvalue>
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




## 格式化namenode
```bash
$HADOOP_HOME/bin/hadoop namenode -format
```
启动服务测试
	yarn 8088端口
	hdfs 50070端口

## spark只需要在slaves中添加
	slave01
	slave02

	**sparkUI端口8080**
## 执行spark on yarn命令行模式
```
spark-shell --master yarn --deploy-mode client --driver-memory 1g --executor-memory 1g --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 512m --executor-memory 512m --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 350m --executor-memory 350m --executor-cores 1

spark-shell --master yarn --deploy-mode client --driver-memory 650m --executor-memory 650m --executor-cores 1
```
# 整个搭建过程中

	首先是参照搭建网卡，创建一个centos虚拟机，在虚拟机的基础上创建了一个docker-machine(非mac可使用pipework的方式）
	引用文章https://github.com/SixQuant/engineering-excellence/blob/master/docker/docker-install-mac-vm-centos.md
	引用的是有ssh服务的Docker镜像**kinogmt/centos-ssh:6.7**，生成容器os
	再在此基础上进行组件的安装，最后保存为镜像centos:hadoop-spark
	参考文章https://blog.csdn.net/GOGO_YAO/article/details/76863201
	创建了master slave01 slave02容器
	yao/os：1.0  拥有sshd服务，并且开机启动，安装了rz vim等 是在yaoos容器基础上保存的建立的  yaoos是一个基础，不能删除

## sz rz与服务器交互上传下载文件

sudo yum install lrzsz -y

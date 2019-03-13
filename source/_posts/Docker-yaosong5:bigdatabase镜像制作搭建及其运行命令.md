---
title:  Docker-yaosong5/bigdatabase镜像制作搭建及其运行命令
date: 2018年08月06日 22时15分52秒
tags:  [Docker]
categories: Docker
toc: true
typora-copy-images-to: ipic
---



[TOC]

## 在centosbase1.0的基础上设置变量

    在centosbase1.0的基础上安装，mysql（mysql安装参考mysql安装文章）里面安装hive用户，hue用户等等，
    创建数据库hue，hive等（为后续做准备）
```
create database hive;
create database hue;
```
    开机自启动mysql
    `chkconfig --add mysqld`

  保存为镜像yaosong5/centosbase:2.0
`docker commit -m "centos ssh mysql 创建了hive库，hue库"  ctos   yaosong5/centosbase:2.0`
`docker commit -m "centos ssh mysql 创建了hive库，hue库"  ctos   yaosong5/centosbase:3.0`
`docker commit -m "centos ssh mysql 创建了hive库，hue库，更改了/JAVA_HOME"  ctos   yaosong5/centosbase:4.0`
后更改为3.0重新配置了java_home
将标签4.0 更改为  centosbase:1.0
还是创建 bigdatabase:1.0





## 准备
创建软连接
```
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hadoop      /Users/yaosong/Yao/share/source/hadoop
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hbase      /Users/yaosong/Yao/share/source/hbase
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hive      /Users/yaosong/Yao/share/source/hive
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/kafka      /Users/yaosong/Yao/share/source/kafka
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/scala      /Users/yaosong/Yao/share/source/scala
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/spark      /Users/yaosong/Yao/share/source/spark
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/zk      /Users/yaosong/Yao/share/source/zk
ln -s /Users/yaosong/Yao/DockerSource/hueSource/ant      /Users/yaosong/Yao/share/source/ant
ln -s /Users/yaosong/Yao/DockerSource/hueSource/hue4      /Users/yaosong/Yao/share/source/hue4
ln -s /Users/yaosong/Yao/DockerSource/hueSource/maven      /Users/yaosong/Yao/share/source/maven
ln -s /Users/yaosong/Yao/DockerSource/flinkSource/flink      /Users/yaosong/Yao/share/source/flink
ln -s /Users/yaosong/Yao/DockerSource/flinkSource/flink-hadoop      /Users/yaosong/Yao/share/source/flink-hadoop
ln -s /Users/yaosong/Yao/DockerSource/elkSource/es      /Users/yaosong/Yao/share/source/es
ln -s /Users/yaosong/Yao/DockerSource/elkSource/filebeat      /Users/yaosong/Yao/share/source/filebeat
ln -s /Users/yaosong/Yao/DockerSource/elkSource/kibana      /Users/yaosong/Yao/share/source/kibana
ln -s /Users/yaosong/Yao/DockerSource/elkSource/logstash      /Users/yaosong/Yao/share/source/logstash
ln -s /Users/yaosong/Yao/DockerSource/elkSource/node      /Users/yaosong/Yao/share/source/node
ln -s /Users/yaosong/Yao/DockerSource/hueSource/jdk      /Users/yaosong/Yao/share/source/jdk

后调整
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/spark-2.3.1-bin-hadoop2.7      /Users/yaosong/Yao/share/source/spark
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hadoop-3.0.3      /Users/yaosong/Yao/share/source/hadoop

rm -rf /Users/yaosong/Yao/share/source/hadoop
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hadoop-3.1.0      /Users/yaosong/Yao/share/source/hadoop

rm -rf /Users/yaosong/Yao/share/source/hadoop
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hadoop-2.9.0      /Users/yaosong/Yao/share/source/hadoop

rm -rf /Users/yaosong/Yao/share/source/hadoop
ln -s /Users/yaosong/Yao/DockerSource/sparkSource/hadoop      /Users/yaosong/Yao/share/source/hadoop

ln -s /Users/yaosong/Yao/DockerSource/elkSource/elasticsearch-head-master    /Users/yaosong/Yao/share/source/elasticsearch-head-master

ln -s  /Users/yaosong/Yao/DockerSource/sparkSource/kafkaall/kafka1 /Users/yaosong/Yao/share/source/
ln -s  /Users/yaosong/Yao/DockerSource/sparkSource/kafkaall/kafka2 /Users/yaosong/Yao/share/source/
ln -s  /Users/yaosong/Yao/DockerSource/sparkSource/kafkaall/kafka3 /Users/yaosong/Yao/share/source/


ln -s /Users/yaosong/Yao/DockerSource/elkSource/es1/ /Users/yaosong/Yao/share/source/
ln -s /Users/yaosong/Yao/DockerSource/elkSource/es2/ /Users/yaosong/Yao/share/source/
ln -s /Users/yaosong/Yao/DockerSource/elkSource/es3/ /Users/yaosong/Yao/share/source/



ln -s /Users/yaosong/Yao/启动脚本/kafka-start-all.sh /Users/yaosong/Yao/share/shell/kafka-start-all.sh
ln -s /Users/yaosong/Yao/启动脚本/es-start.sh /Users/yaosong/Yao/share/shell/es-start.sh
ln -s /Users/yaosong/Yao/启动脚本/filebeat-start.sh /Users/yaosong/Yao/share/shell/filebeat-start.sh
ln -s /Users/yaosong/Yao/启动脚本/hadoop-start.sh /Users/yaosong/Yao/share/shell/hadoop-start.sh
ln -s /Users/yaosong/Yao/启动脚本/hbase-start.sh /Users/yaosong/Yao/share/shell/hbase-start.sh
ln -s /Users/yaosong/Yao/启动脚本/hbasezkStart.sh /Users/yaosong/Yao/share/shell/hbasezkStart.sh
ln -s /Users/yaosong/Yao/启动脚本/header-start.sh /Users/yaosong/Yao/share/shell/header-start.sh
ln -s /Users/yaosong/Yao/启动脚本/hue-start.sh /Users/yaosong/Yao/share/shell/hue-start.sh
ln -s /Users/yaosong/Yao/启动脚本/kibana-start.sh /Users/yaosong/Yao/share/shell/kibana-start.sh
ln -s /Users/yaosong/Yao/启动脚本/logstash-start.sh /Users/yaosong/Yao/share/shell/logstash-start.sh
ln -s /Users/yaosong/Yao/启动脚本/spark-start.sh /Users/yaosong/Yao/share/shell/spark-start.sh
ln -s /Users/yaosong/Yao/启动脚本/zk-start.sh /Users/yaosong/Yao/share/shell/zk-start.sh


ln -s /Users/yaosong/Yao/启动脚本/hivebeeline.sh /Users/yaosong/Yao/share/shell/hivebeeline.sh
ln -s /Users/yaosong/Yao/启动脚本/hivemetastore.sh /Users/yaosong/Yao/share/shell/hivemetastore.sh
ln -s /Users/yaosong/Yao/启动脚本/hiveservers2.sh /Users/yaosong/Yao/share/shell/hiveservers2.sh
ln -s /Users/yaosong/Yao/启动脚本/hivestart.sh /Users/yaosong/Yao/share/shell/hivestart.sh
```





在virtualbox中设置共享文件夹的share名称对应mac的目录
虚拟机中的目录
sudo mount -t vboxsf share  /Users/yaosong/Yao/share/
sudo mount -t vboxsf yaosong  /Users/yaosong 

sudo mount -t vboxsf vagrant /Users/yaosong 





以后相当于操作/Users/yaosong/Yao/share/就是操作的mac系统/Users/yaosong/Yao/share/

创建dockerfile
```
FROM yaosong5/centosbase:3.0
ENV JAVA_HOME=/usr/java/jdk
ENV PATH=$JAVA_HOME/bin:$PATH
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV SCALA_HOME=/usr/scala/
ENV HADOOP_HOME=/usr/hadoop
ENV HADOOP_CONFIG_HOME=$HADOOP_HOME/etc/hadoop
ENV HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
ENV PATH=$PATH:$HADOOP_HOME/bin
ENV PATH=$PATH:$HADOOP_HOME/sbin
ENV SPARK_LOCAL_DIRS=/usr/spark
ENV SPARK_DRIVER_MEMORY=1G
ENV SPARK_HOME=/usr/spark
ENV PATH=$SPARK_HOME/bin:$PATH
ENV PATH=$SPARK_HOME/sbin:$PATH
ENV MAVEN_HOME=/usr/maven
ENV PATH=${PATH}:${MAVEN_HOME}/bin
ENV HIVE_HOME=/usr/hive
ENV PATH=$HIVE_HOME/bin:$PATH
ENV KAFKA_HOME=/usr/kafka
ENV PATH=$KAFKA_HOME/bin:$PATH
ENV ANT_HOME=/usr/ant
ENV PATH=$JAVA_HOME/bin:$ANT_HOME/bin:$PATH
ENV ANT_HOME PATH
ENV HUE_HOME=/usr/hue4
ENV ZK_HOME=/usr/zk
ENV HBASE_HOME=/usr/hbase
ENV PATH=$HBASE_HOME/bin:$PATH
ENV PATH=$ZK_HOME/bin:$PATH
#elk
ENV ES_HOME=/usr/es
ENV PATH=$ES_HOME/bin:$PATH
ENV KIBANA_HOME=/usr/kibana
ENV PATH=$KIBANA_HOME/bin:$PATH
ENV LOGSTASH_HOME=/usr/logstash
ENV PATH=$LOGSTASH_HOME/bin:$PATH
ENV NODE_HOME=/usr/node-v6.12.0-linux-x64
ENV PATH=$NODE_HOME/bin:$PATH
ENV NODE_PATH=$NODE_HOME/lib/node_modules
#flink
ENV FLINK_HOME=/usr/flink
ENV PATH=$FLINK_HOME/bin:$PATH
```
### 构建镜像
`docker build -f Dockerfile -t yaosong5/bigdatabase:1.0 .`

## 分别构建容器和镜像 （采用挂载的方式）
分别的启动命令
### 先加载后启动一个hadoop集群
#### 创建一个master容器
不含启动参数，只是用原始的配置文件，只启动容器，不使用任何命令启动组件

由于下面三个组件需要挂载相同的目录那么创建一个共享的数据卷镜像

参考 ：https://blog.csdn.net/magerguo/article/details/72514813
https://blog.csdn.net/u013571243/article/details/51751367

docker run  -it --rm --name datavol  -h datavol   -v  /Users/yaosong/Yao/share/source/:/usr/   busybox:latest /bin/sh
docker run  -it --rm --name datavol  -h datavol   -v  /Users/yaosong/Yao/share/source/:/usr/  busybox:latest /bin/sh
rm详解http://dockone.io/article/152    https://blog.csdn.net/taiyangdao/article/details/73076770

```
docker run  -itd --name datavol  -h datavol   \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop \
-v /Users/yaosong/Yao/share/data/hadoop/name:/usr/hadoop/namenode \
busybox:latest /bin/sh
```



> ### 数据容器
>
> 常见的使用场景是使用纯数据容器来持久化数据库、配置文件或者数据文件等。
>
> 官方的文档
>
> 上有详细的解释。例如：
>
> ```
> $ docker run --name dbdata postgres echo "Data-only container for postgres"
>
> ```
>
> 该命令将会创建一个已经包含在 Dockerfile 里定义过 Volume 的 postgres 镜像，运行
>
> ```
> echo
> ```
>
> 命令然后退出。当我们运行
>
> ```
> docker ps
> ```
>
> 命令时，
>
> ```
> echo
> ```
>
> 可以帮助我们识别某镜像的用途。我们可以用
>
> ```
> -volumes-from
> ```
>
> 使用数据容器的两个注意点：
>
> - 不要运行数据容器，这纯粹是在浪费资源。
>
> - 不要为了数据容器而使用 “最小的镜像”，如`busybox`或`scratch`，只使用数据库镜像本身就可以了。你已经拥有该镜像，所以并不需要占用额外的空间。
>
>   参考http://dockone.io/article/128

再使用 --volumes-from  datavol
示例 ：docker run -it  --net=br  --volumes-from dataVol ubuntu64 /bin/bash

####  使用公用挂载镜像

**创建基础镜像**

```
docker run  -itd --name datavol  -h datavol   \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop \
-v /Users/yaosong/Yao/share/data/hadoop/name:/usr/hadoop/namenode \
-v /Users/yaosong/Yao/share/source/spark:/usr/spark \
-v /Users/yaosong/Yao/share/source/flink:/usr/flink \
-v /Users/yaosong/Yao/share/source/ant:/usr/ant \
-v /Users/yaosong/Yao/share/source/maven:/usr/maven \
busybox:latest /bin/sh
```



##### master slave01 slave02
```shell
docker run -itd --net=br  --name master -h master --volumes-from  datavol \
-v /Users/yaosong/Yao/share/shell/hadoop-start.sh:/hadoop-start.sh yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd --name slave01 --net=br -h slave01 --volumes-from  datavol \
-v /Users/yaosong/Yao/share/data/hadoop/data1:/usr/hadoop/datanode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop  yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd --name slave02 --net=br -h slave02 --volumes-from  datavol \
-v /Users/yaosong/Yao/share/data/hadoop/data2:/usr/hadoop/datanode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop yaosong5/bigadatabase:1.0 &> /dev/null



docker run -itd --net=br  --name master -h master --volumes-from  datavol \
-v /Users/yaosong/Yao/share/shell/hadoop-start.sh:/hadoop-start.sh yaosong5/centosbase:1.0 &> /dev/null

docker run -itd --name slave01 --net=br -h slave01 --volumes-from  datavol \
-v /Users/yaosong/Yao/share/data/hadoop/data1:/usr/hadoop/datanode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop  yaosong5/centosbase:1.0 &> /dev/null

docker run -itd --name slave02 --net=br -h slave02 --volumes-from  datavol \
-v /Users/yaosong/Yao/share/data/hadoop/data2:/usr/hadoop/datanode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop yaosong5/centosbase:1.0 &> /dev/null
```



**创建大节点** 使用的1.1封装了hue 1.1就是2.0

```bash
docker run -itd --net=br  --name master -h master --volumes-from  datavol \
-v /Users/yaosong/Yao/share/shell/spark-start.sh:/spark-start.sh \
-v /Users/yaosong/Yao/share/shell/hadoop-start.sh:/hadoop-start.sh \
-v /Users/yaosong/Yao/share/source/hue4/desktop/conf/hue.ini:/usr/hue4/desktop/conf/hue.ini \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd --name slave01 --net=br -h slave01 --volumes-from  datavol \
-v /Users/yaosong/Yao/share/hadoop/data1:/usr/hadoop/datanode \
 yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd --name slave02 --net=br -h slave02 --volumes-from  datavol \
-v /Users/yaosong/Yao/share/hadoop/data2:/usr/hadoop/datanode \
yaosong5/bigadatabase:1.0 &> /dev/null
```





#### 原生容器自身挂载镜像
##### master slave01 slave02
```shell
docker run -itd --net=br  --name master -h master \
-v /Users/yaosong/Yao/share/data/hadoop/name:/usr/hadoop/namenode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop \
-v /Users/yaosong/Yao/share/shell/hadoop-start.sh:/hadoop-start.sh yaosong5/bigadatabase:1.0 &> /dev/null


docker run -itd --name slave01 \
--net=br -h slave01 \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop \
-v /Users/yaosong/Yao/share/data/hadoop/name:/usr/hadoop/namenode \
-v /Users/yaosong/Yao/share/data/hadoop/data1:/usr/hadoop/datanode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop  yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name slave02 \
--net=br -h slave02 \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop \
-v /Users/yaosong/Yao/share/data/hadoop/name:/usr/hadoop/namenode \
-v /Users/yaosong/Yao/share/data/hadoop/data2:/usr/hadoop/datanode \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop yaosong5/bigadatabase:1.0 &> /dev/null

```

##### zk
/Users/yaosong/Yao/share/data/zk/zk1/data/myid
/Users/yaosong/Yao/share/data/zk/zk2/data/myid
/Users/yaosong/Yao/share/data/zk/zk3/data/myid
/Users/yaosong/Yao/share/shell/zk-start.sh

```bash
docker run -itd  --name zk1 --net=br -h zk1 \
-v /Users/yaosong/Yao/share/source/zk:/usr/zk \
-v /Users/yaosong/Yao/share/shell/zk-start.sh:/zk-start.sh \
-v /Users/yaosong/Yao/share/data/zk/zk1/data/myid:/usr/zk/data/myid \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name zk2 --net=br -h zk2 \
-v /Users/yaosong/Yao/share/source/zk:/usr/zk \
-v /Users/yaosong/Yao/share/data/zk/zk2/data/myid:/usr/zk/data/myid \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name zk3 --net=br -h zk3 \
-v /Users/yaosong/Yao/share/source/zk:/usr/zk \
-v /Users/yaosong/Yao/share/data/zk/zk3/data/myid:/usr/zk/data/myid \
yaosong5/bigadatabase:1.0 &> /dev/null
```
##### hbase
```Bash
docker run -itd  --name hbase1 --net=br -h hbase1 \
-v /Users/yaosong/Yao/share/source/hbase:/usr/hbase \
-v /Users/yaosong/Yao/share/shell/hbase-start.sh:/hbase-start.sh \
-v /Users/yaosong/Yao/share/shell/hbasezkStart.sh:/hbasezkStart.sh \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name hbase2 --net=br -h hbase2 \
-v /Users/yaosong/Yao/share/source/hbase:/usr/hbase \
-v /Users/yaosong/Yao/share/shell/hbase-start.sh:/hbase-start.sh \
-v /Users/yaosong/Yao/share/shell/hbasezkStart.sh:/hbasezkStart.sh \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name hbase3 --net=br -h hbase3 \
-v /Users/yaosong/Yao/share/source/hbase:/usr/hbase \
-v /Users/yaosong/Yao/share/shell/hbase-start.sh:/hbase-start.sh \
-v /Users/yaosong/Yao/share/shell/hbasezkStart.sh:/hbasezkStart.sh \
yaosong5/bigadatabase:1.0 &> /dev/null
```

##### hive
```bash
docker run -itd  --name hive --net=br -h hive --volumes-from  hivedatavol  \
-v /Users/yaosong/Yao/share/source/hive:/usr/hive  \
-v /Users/yaosong/Yao/share/shell/hive-start-metastore.sh:/hive-start-metastore.sh  \
-v /Users/yaosong/Yao/share/shell/hive-start-servers2.sh:/hive-start-servers2.sh  \
-v /Users/yaosong/Yao/share/shell/hive-start.sh:/hive-start.sh  \
-v /Users/yaosong/Yao/share/shell/hivebeeline.sh:/hivebeeline.sh  \
 yaosong5/bigadatabase:1.0  &> /dev/null
 
 docker run -itd  --name hive --net=br -h hive  \
-v /Users/yaosong/Yao/share/source/hive:/usr/hive  \
-v /Users/yaosong/Yao/share/source/hadoop:/usr/hadoop  \
-v /Users/yaosong/Yao/share/shell/hive-start-metastore.sh:/hive-start-metastore.sh  \
-v /Users/yaosong/Yao/share/shell/hive-start-servers2.sh:/hive-start-servers2.sh  \
-v /Users/yaosong/Yao/share/shell/hive-start.sh:/hive-start.sh  \
-v /Users/yaosong/Yao/share/shell/hivebeeline.sh:/hivebeeline.sh  \
 yaosong5/bigadatabase:1.0  &> /dev/null
```

##### kafka



/Users/yaosong/Yao/share/shell/kafka-start-all.sh

/Users/yaosong/Yao/share/source/kafka

```
docker run -itd  --name kafka1 --net=br -h kafka1 \
-v /Users/yaosong/Yao/share/source/kafka:/usr/kafka \
-v /Users/yaosong/Yao/share/shell/kafka-start-all.sh:/kafka-start-all.sh \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name kafka2 --net=br -h kafka2 \
-v /Users/yaosong/Yao/share/source/kafka:/usr/kafka \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name kafka3 --net=br -h kafka3 \
-v /Users/yaosong/Yao/share/source/kafka:/usr/kafka \
yaosong5/bigadatabase:1.0 &> /dev/null
```

## spark

```shell
docker run -itd  --name spark1 --net=br -h spark1 --volumes-from  datavol  \
-v /Users/yaosong/Yao/share/source/spark:/usr/spark \
-v /Users/yaosong/Yao/share/shell/spark-start.sh:/spark-start.sh \
yaosong5/bigadatabase:1.0 &> /dev/null

docker run -itd  --name spark2 --net=br -h spark2 --volumes-from  datavol  \
-v /Users/yaosong/Yao/share/source/spark:/usr/spark \
yaosong5/bigadatabase:1.0 &> /dev/null


docker run -itd  --name spark3 --net=br -h spark3 --volumes-from  datavol  \
-v /Users/yaosong/Yao/share/source/spark:/usr/spark \
yaosong5/bigadatabase:1.0 &> /dev/null
```

*用* &> /dev/null这种方式sshd服务才会启动

停止 and 删除容器

```bash
docker start master
docker start slave01
docker start slave02

docker stop master
docker stop slave01
docker stop slave02
docker rm master
docker rm slave01
docker rm slave02
docker stop datavol && docker rm datavol


docker stop hive
docker rm hive

docker stop zk1
docker stop zk2
docker stop zk3
docker rm zk1
docker rm zk2
docker rm zk3


docker stop hbase1
docker stop hbase2
docker stop hbase3
docker rm hbase1
docker rm hbase2
docker rm hbase3



docker stop kafka1
docker stop kafka2
docker stop kafka3
docker rm kafka1
docker rm kafka2
docker rm kafka3

docker start spark1
docker start spark2
docker start spark3

docker stop spark1
docker stop spark2
docker stop spark3
docker rm spark1
docker rm spark2
docker rm spark3


docker stop hadoop1
docker stop hadoop2
docker stop hadoop3
docker rm hadoop1
docker rm hadoop2
docker rm hadoop3


docker stop elk1
docker stop elk2
docker stop elk3
docker rm elk1
docker rm elk2
docker rm elk3
```



## compose编排



说的








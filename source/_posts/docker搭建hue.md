---
title:  Docker搭建hue
date: 2018年06月21日 22时15分52秒
tags:  [Docker,Hue]
categories: 部署安装
toc: true
---
[toc]
![enter description here](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/7/20/1532025321846.jpg)
# 本次采用的ant maven来编译hue

## 启动一个基础容器
`docker run -itd  --net=br  --name hue --hostname  hue yaosong5/centosbase:1.0 &> /dev/null`
<!--more-->
## 拷贝源包
将ant、hue4.0.0、ant、maven等下载到本地结业后，再拷贝到容器（这样更快速）

	docker cp  /Users/yaosong/Yao/ant   4115ea59088e:/
	docker cp  /Users/yaosong/Yao/maven  4115ea59088e:/
	docker cp  /Users/yaosong/Yao/hue4  4115ea59088e:/usr/

## 配置HOME
```bash
vim ~/.bashrc
加入
MAVEN_HOME=/maven
export MAVEN_HOME
export PATH=${PATH}:${MAVEN_HOME}/bin
ANT_HOME=/ant
PATH=$JAVA_HOME/bin:$ANT_HOME/bin:$PATH
HUE_HOME=/hue4
使其生效
source ~/.bashrc
```
## 安装依赖，编译hue需要安装一些依赖

`yum install gmp-devel -y `
> 参考 http://www.aizhuanji.com/a/0Vo0qEMW.html

**若解决不了**
```
yum install asciidoc cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libtidy libxml2-devel libxslt-devel make mysql-devel openldap-devel  sqlite-devel openssl-devel gmp-devel -y
```

> 参考链接：https://www.jianshu.com/p/417788238e3d

## 编译安装hue

### 首先编译 Hue，并在要安装 Hue 的节点上创建 Hue 用户和 hue 组


创建 Hue 用户
```
groupadd hue
useradd hue -g hue
cd   $HUE_HOME
chown -R hue:hue *
```

> 注：需要注意的是 hue 在编译时有两种方式:1.通过maven、ant编译 2.通过python编译（在centos6.5因为自身python为2.6.6版本和hue编译需要2.7版本会有一点小冲突，故采用1）两种方式都是在hue目录下 make apps，只是第一种方式要先配置maven、ant的环境而已

```bash
cd $HUE_HOME
make apps
```
> 参考 ：https://blog.csdn.net/u012802702/article/details/68071244

### 安装mysql
由于需要hue需要存放一些元数据故安装mysql
```bash
yum install -y mysql-server
service mysqld start
mysql -u root -p
Enter password:           //默认密码为空，输入后回车即可
set password for root@localhost=password('root'); 　　密码设置为root
默认情况下Mysql只允许本地登录，所以只需配置root@localhost就好
set password for root@%=password('root'); 　　　　　　密码设置为root （其实这一步可以不配）
set password for root@master=password('root'); 　　密码设置为root （其实这一步可以不配）

select user,host,password from mysql.user;  　　查看密码是否设置成功
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;

create database hue;
```

### 更改hue的配置文件（关于hadoop，hive等）
`vim $HUE_HOME/desktop/conf/hue.ini`

	集成 hive
	hive_server_host=master
	hive_server_port=10000
	hive_conf_dir=$HIVE_HOME/conf


	集成 hadoop
	fs_defaultfs=hdfs://master:9000
	logical_name=master
	webhdfs_url=http://master:50070/webhdfs/v1
	hadoop_hdfs_home=$HADOOP_HOME
	hadoop_conf_dir=$HADOOP_HOME/etc/hadoop

	配置 yarn
	在 [hadoop].[[yarn_clusters]].[[[default]]] 下
	resourcemanager_host=master
	resourcemanager_port=8032
	resourcemanager_api_url=http://master:8088
	proxy_api_url=http://master:8088


	集成 hbase
	在 [hbase] 节点下
	hbase_clusters=(HBASE|master:9090)
	hbase_conf_dir=$HBASE_HOME/conf


>[参考]https://blog.csdn.net/u012802702/article/details/68071244
### 解决 hue ui 界面查询中文乱码问题
在 `[[[mysql]]] `节点下

<kbd>options={"init_command":"SET NAMES'utf8'"}</kbd>
> [参考]
https://blog.csdn.net/u012802702/article/details/68071244

### 初始化mysql

完成以上的这个配置，启动 Hue, 通过浏览器访问，会发生错误，原因是 mysql 数据库没有被初始化
DatabaseError: (1146,"Table 'hue.desktop_settings' doesn't exist")
执行以下指令对 hue 数据库进行初始化

	cd $HUE_HOME/build/env/
	bin/hue syncdb
	bin/hue migrate

此外需要注意的是如果使用的是：
$HUE_HOME/build/env/bin/hue syncdb --noinput

则不会让输入初始的用户名和密码，只有在首次登录时才会让输入，作为超级管理员账户。

## 需要对大数据各组件进行满足hue进行相应配置和启动

### hdfs

#### hdfs-site.xml
增加一个值开启 hdfs 的 web 交互
```xml
 <!--HUE 增加一个值开启 hdfs 的 web 交互-->
<property>
      <name>dfs.webhdfs.enabled</name>
      <value>true</value>
</property>
 <!--HUE 增加一个值开启 hdfs 的 web 交互-->
```

#### core-site.xml
```xml
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

### hbase
hue 访问 hbase 是用的 thriftserver，并且是 thrift1，不是 thrift2，所以要在 master 上面启动 thrif1

```
$HBASE_HOME/bin/hbase-daemon.sh start thrift
```
> 参考 https://blog.csdn.net/Dante_003/article/details/78889084

### hive
Hue 与框架 Hive 的集成
开启 Hive Remote MetaStore
`nohup $HIVE_HOME/bin/hive --service metastore &`

hive 只需启动 hiveserver2，thriftserver 的 10000 端口启动即可
```bash
nohup $HIVE_HOME/bin/hiveserver2 &
或者
nohup HIVE_HOME/bin/hive --service hiveserver2 &
```

## 依赖的组件启动


	首先启动 hadoop
	start-all.sh


	然后需要同时启动 hive 的 metastore 和 hiveserve2
	nohup hive --service metastore &
	nohup hive --service hiveserver2 &

	Hue 需要读取 HBase 的数据是使用 thrift 的方式，默认 HBase 的 thrift 服务没有开启，所有需要手动额外开启 thrift 服务。
	thrift service 默认使用的是 9090 端口，使用如下命令查看端口是否被占用
	netstat -nl|grep 9090

	启动 thrift service
	$HBASE_HOME/bin/hbase-daemon.sh start thrift


	build/env/bin/hue runserver 192.168.200.200:8181
	浏览器输入 192.168.200.200:8181 可进入 hue 界面


> 【参考】https://blog.csdn.net/hexinghua0126/article/details/80338779

	异常：
	如果修改配置文件后，启动后无法进人 hue 界面，可能是配置文件被锁住了，或者 hadoop、hive 等服务没有启动起来
	cd $HUE_HOME/desktop/conf
	ls –a
	rm –rf hue.ini.swp

在 hue界面异常，导致 hive 无法使用
安装插件：
yum install cyrus-sasl-plain  cyrus-sasl-devel  cyrus-sasl-gssapi

#### 依赖启动的脚本

```bash
#!/bin/bash
#启动mysql
service mysqld start
#启动hadoop
sh /hadoop-start.sh
#启动hive
sh /hive-start-servers2.sh
#启动 thrift service
$HBASE_HOME/bin/hbase-daemon.sh start thrift
#启动hue
nohup $HUE_HOME/build/env/bin/supervisor &
```


## hue启动命令
```
$HUE_HOME/build/env/bin/supervisor
```

(注：想要后台执行就是 **$HUE_HOME/build/env/bin/supervisor &** , 停止是**pkill -U hue**)

**hue的web服务端口：8888**

## 保存为镜像
`docker commit -m "hue"  hue   yaosong5/hue4:1.0`

## 创建容器
`docker run -itd  --net=br  --name gethue --hostname gethue gethue/hue:latest &> /dev/null`
映射宿主机的hosts文件及其hue的配置文件方式启动容器
```
docker run --name=hue -d --net=br
 -v /etc/hosts/:/etc/hosts -v $PWD/pseudo-distributed.ini:/hue/desktop/conf/pseudo-distributed.ini 	yaosong5/hue4:1.0
```

--net=br为了宿主机和容器之前ip自由访问所搭建的网络模式，如有需求请参考

**其他参考**

```bash
docker run --name=hue -d --net=br -v /etc/hosts/:/etc/hosts -v $PWD/pseudo-distributed.ini:/hue/desktop/conf/pseudo-distributed.ini gethue/hue:latest
#注意：
#让hue在master节点上启动，例如hdfs、yarn、regionmaster、hive等这些节点上，这样容器里面的hue可以直接使用本机的配置文件和服务
#--net=host，使用宿主机的网络
#将本机的hosts映射进去替换，如hdfs上传的时候，要用到datanode的hostname
```
> 参考：https://blog.csdn.net/Dante_003/article/details/78889084

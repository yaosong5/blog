---
title:  Hue搭建
date: 2018年06月21日 22时15分52秒
tags:  [Docker,Hue]
categories: 安装部署
toc: true
---
[TOC]
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

首先编译 Hue，并在要安装 Hue 的节点上创建 Hue 用户和 hue 组




创建 Hue 用户
```Bash
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

如果报错

```
/usr/hue4/Makefile.vars:42: *** "Error: must have python development packages for 2.6 or 2.7. Could not find Python.h. Please install python2.6-devel or python2.7-devel".  Stop.
```

需执行

```bash
可以先查看一下含 python-devel 的包
yum search python | grep python-devel
64 位安装 python-devel.x86_64，32 位安装 python-devel.i686，我这里安装:
sudo yum install python-devel.x86_64 -y
```

## 更改hue的配置文件

`vim $HUE_HOME/desktop/conf/hue.ini`

### mysql

找到位置更改host

### hive

```ini
hive_server_host=master
hive_server_port=10000
hive_conf_dir=$HIVE_HOME/conf
```

### hadoop-hdfs

```ini
fs_defaultfs=hdfs://master:9000
logical_name=master
webhdfs_url=http://master:50070/webhdfs/v1
hadoop_hdfs_home=$HADOOP_HOME
hadoop_conf_dir=$HADOOP_HOME/etc/hadoop
```



### hadoop-yarn

在 [hadoop].[[yarn_clusters]].[[[default]]] 下

```ini
resourcemanager_host=master
resourcemanager_port=8032
resourcemanager_api_url=http://master:8088
proxy_api_url=http://master:8088
```

### hbase


在 [hbase] 节点下

```ini
hbase_clusters=(HBASE|master:9090)
hbase_conf_dir=$HBASE_HOME/conf
use_doas=true
```



## 大数据各组件满足hue进行相应配置

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

### 报错DatabaseError

> DatabaseError:(1146,"Table 'hue.desktop_settings' doesn't exist")-初始化mysql

完成以上的这个配置，启动 Hue, 通过浏览器访问，会发生错误，原因是 mysql 数据库没有被初始化
**DatabaseError: (1146,"Table 'hue.desktop_settings' doesn't exist")**
执行以下指令对 hue 数据库进行初始化

```bash
cd $HUE_HOME/build/env/
bin/hue syncdb
bin/hue migrate
```

此外需要注意的是如果使用的是：
`$HUE_HOME/build/env/bin/hue syncdb --noinput`

则不会让输入初始的用户名和密码，只有在首次登录时才会让输入，作为超级管理员账户。

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

**hbase-site.xml**

```xml
<!-- hue支持 -->
<property>
        <name>hbase.thrift.support.proxyuser</name>
        <value>true</value>
</property>
<property>
        <name>hbase.regionserver.thrift.http</name>
        <value>true</value>
</property>
<!-- hue支持 -->
```

hue 访问 hbase 是用的 thriftserver，并且是 thrift1，不是 thrift2，所以要在 master 上面启动 thrif1

```
$HBASE_HOME/bin/hbase-daemon.sh start thrift
```
> 参考 https://blog.csdn.net/Dante_003/article/details/78889084

### 读取hbase问题

为解决访问
**Failed to authenticate to HBase Thrift Server, check authentication configurations.**
需要在hue的配置文件中配置

```ini
use_doas=true
```


参考http://gethue.com/hbase-browsing-with-doas-impersonation-and-kerberos/

若以上配置未能解决问题，还需要将core-site.xml拷贝到hbase/conf，并添加以下内容

```xml
<property>
  <name>hadoop.proxyuser.hbase.hosts</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.hbase.groups</name>
  <value>*</value>
</property>
```



> [参考]https://blog.csdn.net/u012802702/article/details/68071244

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

### 解决 hue ui 界面查询中文乱码问题

在 `[[[mysql]]] `节点下

```ini
options={"init_command":"SET NAMES'utf8'"}
```

> [参考]
> https://blog.csdn.net/u012802702/article/details/68071244



### 

## 依赖的组件启动

### Mysql

`service mysqld start`

### hadoop

`start-all.sh`

### hive

然后需要同时启动 hive 的 metastore 和 hiveserve2

```bash
nohup hive --service metastore &
nohup hive --service hiveserver2 &
```

### hbase

Hue 需要读取 HBase 的数据是使用 thrift 的方式，默认 HBase 的 thrift 服务没有开启，所有需要手动额外开启 thrift 服务。

启动 thrift service
`$HBASE_HOME/bin/hbase-daemon.sh start thrift`

thrift service 默认使用的是 9090 端口，使用如下命令查看端口是否被占用

`netstat -nl|grep 9090`

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
```$HUE_HOME/build/env/bin/supervisor &```

(注：想要后台执行就是 **$HUE_HOME/build/env/bin/supervisor &** )

或者

`$HUE_HOME/build/env/bin/hue runserver_plus 0.0.0.0:8888`

>
> 【参考】https://blog.csdn.net/hexinghua0126/article/details/80338779
>

**hue的web服务端口：8888*

## hue停止命令

`pkill -U hue`

# 报错

1、如果修改配置文件后，启动后无法进人 hue 界面

```
可能是配置文件被锁住了
cd $HUE_HOME/desktop/conf
ls –a
rm –rf hue.ini.swp
或者 hadoop、hive 等服务没有启动起来
```

2、在 hue界面异常，导致 hive 无法使用
安装插件：
`yum install cyrus-sasl-plain  cyrus-sasl-devel  cyrus-sasl-gssapi`

# 操作镜像

## 保存为镜像
`docker commit -m "hue"  hue   yaosong5/hue4:1.0`

## 创建容器
`docker run -itd  --net=br  --name gethue --hostname gethue gethue/hue:latest &> /dev/null`
映射宿主机的hosts文件及其hue的配置文件方式启动容器
```
docker run --name=hue -d --net=br
 -v /etc/hosts/:/etc/hosts -v $PWD/pseudo-distributed.ini:/hue/desktop/conf/pseudo-distributed.ini 	yaosong5/hue4:1.0
```

--net=br为了宿主机和容器之前ip自由访问所搭建的网络模式，如有需求请参考

**其他参考**

```bash
docker run --name=hue -d --net=br -v /etc/hosts/:/etc/hosts -v $PWD/pseudo-distributed.ini:/hue/desktop/conf/pseudo-distributed.ini gethue/hue:latest
```
> 参考：https://blog.csdn.net/Dante_003/article/details/78889084

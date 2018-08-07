---
title:  HIVE搭建
date: 2018年08月06日 22时15分52秒
tags:  [Docker,HIVE]
categories: 部署安装
toc: true
typora-copy-images-to: ipic
---

[TOC]
# 配置HOME

下载hive包，并解压

```
http://archive.apache.org/dist/
```

`ln -s hive-2.1.1  /usr/hive`


**vi ~/.bashrc**

```bash
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
$HIVE_HOME/bin/hive
export HIVE_HOME=/usr/hive
PATH=$HIVE_HOME/bin:$PATH
#hive依赖于hadoop(可以不运行在同一主机，但是需要hadoop的配置)
$HADOOP_HOME=/usr/hadoop
```

**source ~/.bashrc**

<!--more -->

# 安装mysql

> ## Hive元数据介绍
>
> Hive 将元数据存储在 RDBMS 中，一般常用 MySQL 和 Derby。默认情况下，Hive 元数据保存在内嵌的 Derby 数据库中，只能允许一个会话连接，只适合简单的测试。实际生产环境中不适用， 为了支持多用户会话，则需要一个独立的元数据库，使用 MySQL 作为元数据库，Hive 内部对 MySQL 提供了很好的支持，配置一个独立的元数据库

```bash
yum install -y mysql-server
chkconfig --add mysqld
chkconfig mysqld on
chkconfig --list mysqld
service mysqld start

mysql -u root -p
Enter password:           //默认密码为空，输入后回车即可
set password for root@localhost=password('root'); 　　密码设置为root
set password for root@=password('root');
默认情况下Mysql只允许本地登录，所以只需配置root@localhost就好
设置所有ip访问密码为root
set password for root@%=password('root'); 　　　　　　密码设置为root （其实这一步可以不配）
设置master访问密码为root
set password for root@master=password('root'); 　　密码设置为root （其实这一步可以不配）
查询密码
select user,host,password from mysql.user;  　　查看密码是否设置成功
设置所有ip可以通过root访问
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%' IDENTIFIED BY 'hive' WITH GRANT OPTION;

mysql -uroot -proot
create user 'hive' identified by 'hive';
create user 'hive'@'%' identified by 'hive';

create database hive;
```




# 配置Hive

`mkdir iotmp`

`cp hive-default.xml.template hive-site.xml`

`vim hive-site.xml`
```xml
<configuration>
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
</property>

<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/hive?characterEncoding=UTF-8</value>
    <description>JDBC connect string for a JDBC metastore</description>
</property>

<property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>Username to use against metastore database</description>
</property>

<property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>root</value>
    <description>password to use against metastore database</description>
</property>
<property>
    <name>hive.querylog.location</name>
    <value>/usr/hive/iotmp</value>
    <description>Location of Hive run time structured log file</description>
</property>

<property>
    <name>hive.exec.local.scratchdir</name>
    <value>/usr/hive/iotmp</value>
    <description>Local scratch space for Hive jobs</description>
</property>

<property>
    <name>hive.downloaded.resources.dir</name>
    <value>/usr/hive/iotmp</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
</property>
<property>
	<name>hive.metastore.uris</name>
	<value>thrift://master:9083</value>
</property>
</configuration>
```
Hive 的元数据可以存储在本地的 MySQL 中，但是大多数情况会是一个 mysql 集群，而且不在本地。所以在 hive 中需要开启远程 metastore。由于我是本地的 mysql，我就不配置下列属性了，但是如果是远程的 metastore，配置下面的属性。
```xml
<property>
      <name>hive.metastore.uris</name>
      <value></value>
  <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
</property>
<property>
      <name>hive.server2.transport.mode</name>
      <value>http</value>
      <description>Server transport mode. "binary" or "http".</description>
</property>

链接：https://www.jianshu.com/p/87b76a686216
```

# Hive命令

## 启动hiveserver2

```bash
$HIVE_HOME/bin/hive --service hiveserver2
```

> hiveserver端口号默认是10000

**hiveserver2是否启动**
`netstat -nl|grep 10000`

## 启动hive

```bash
$HIVE_HOME/bin/hive
如果调试，可以加上参数
$HIVE_HOME/bin/hivehive -hiveconf hive.root.logger=DEBUG,console
```

## beeline工具测试使用jdbc方式连接

```bash
$HIVE_HOME/bin/beeline -u jdbc:hive2://localhost:10000
```

使用beeline通过jdbc连接上之后就可以像client一样操作。

hiveserver2会同时启动一个webui，端口号默认为10002，可以通过http://localhost:10002/访问
界面中可以看到Session/Query/Software等信息。(此网页只可查看，不可以操作hive数据仓库)

> 参考https://blog.csdn.net/lblblblblzdx/article/details/79760959
>
> 参考https://www.cnblogs.com/netuml/p/7841387.html





# 报错

> 报错 Hive 2.3.3 MetaException(message:Version information not found in metastore.)

```
schematool -initSchema -dbType mysql
```

> 参考https://stackoverflow.com/questions/50230515/hive-2-3-3-metaexceptionmessageversion-information-not-found-in-metastore
>
> http://sishuok.com/forum/blogPost/list/6221.html
>
> https://blog.csdn.net/nokia_hp/article/details/79054079




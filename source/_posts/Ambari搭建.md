---
title:  Ambari镜像搭建
date: 2018年08月06日 22时15分52秒
tags:  [Docker,Ambari]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---


[TOC]

# 创建基础容器

```bash
docker run -itd  --net=br  --name ambari-agent --hostname  ambari-agent yaosong5/centosbase:1.0 &> /dev/null
```

关闭 selinux , 需要重启
`vim /etc/selinux/config` 

```
SELINUX=disabled
```

<!--more -->

# server端

## 更换yum源

```
wget http://public-repo-1.hortonworks.com/ambari/centos6/2.x/updates/2.0.1/ambari.repo
cp ambari.repo /etc/yum.repos.d
```

## 安装依赖及其server

```
yum install epel-release 
yum repolist
yum install ambari-server  
```

## 启动初始化

`ambari-server setup`
会有一连串的提示

会提示安装 jdk，网速好的可以确定，否则可以下载 jdk-6u31-linux-x64.bin，放到 /var/lib/ambari-server/resources/ 下面，可以指定已经安装的jdk
接着会提示配置用的数据库，可以选择 Oracle 或 postgresql，选择 n 会按默认配置
数据库类型：postgresql
数据库：ambari
用户名：ambari
密码：bigdata
如果提示 Oracle JDK license，yes
等待安装完成



# agent端

安装 ambari-agent

```
yum install -y ambari-agent
chkconfig --add ambari-agent
```

将 ambari.server 上的 3 个. repo 文件复制到 hadoop 集群的三台服务器上；并完成 yum 源更新的命令。

 安装 ambari-agent：在集群的 3 台电脑上执行添加，并添加成开机自启动服务：　　

 yum install -y ambari-agent
 chkconfig --add ambari-agent
 sudo ambari-agent start

# 分别启动server agent 

在server和agent上分别执行

```
ambari-agent start
ambari-server start  
```

## 访问

http://192.168.1.133:8080  

用户名密码: admin,admin



# 保存容器为镜像

```bash
docker commit -m "bigdata:ambari-server"  --author="yaosong"  ambr  yaosong5/ambari-server:1.0
```

# 根据镜像创建容器

```bash
docker run -itd  --net=br  --name ambari1 --hostname ambari1 yaosong5/ambari-server:1.0 &> /dev/null
docker run -itd  --net=br  --name ambari2 --hostname ambari2 yaosong5/ambari-server:1.0 &> /dev/null
docker run -itd  --net=br  --name ambari3 --hostname ambari3 yaosong5/ambari-server:1.0 &> /dev/null
```

# 停止and删除容器

```bash
docker stop ambari1
docker stop ambari2
docker stop ambari3

docker rm ambari1
docker rm ambari2
docker rm ambari3
```
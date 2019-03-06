---
title:  Docker-Hadoop集群【引用】
date: 2018年06月03日 14时36分28秒
tags:  [Docker,Hadoop]
categories: 环境配置
toc: true
---
 Docker配置Hadoop集群环境
 在网上找到一个网友自制的镜像，拉取配置都是参考的，记录一下。
<!--more-->
# 拉取镜像
>sudo docker pull kiwenlau/hadoop-master:0.1.0
>sudo docker pull kiwenlau/hadoop-slave:0.1.0
>sudo docker pull kiwenlau/hadoop-base:0.1.0
>sudo docker pull kiwenlau/serf-dnsmasq:0.1.0


## 查看下载的镜像
![下载的镜像](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/3/1528038081307.jpg)




> sudo docker images

# 在github中拉取源代码
(或者在oschina中拉取)
git clone https://github.com/kiwenlau/hadoop-cluster-docker
开源中国
git clone http://git.oschina.net/kiwenlau/hadoop-cluster-docker


# 运行容器
拉取镜像后，打开源代码文件夹，并且运行脚本
>cd hadoop-cluster-docker


![源包下文件](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/3/1528038513960.jpg)

注意：运行脚本时,需要先启动docker服务
>./start-container.sh
>![enter description here](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/3/1528039457334.jpg)

一共开启了 3 个容器，1 个 master, 2 个 slave。开启容器后就进入了 master 容器 root 用户的根目录（/root）

## 查看root目录下文件
![enter description here](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/3/1528039642562.jpg)

## 测试容器是否正常运行
```serf members```
<!-- more -->

--------
参考：[基于 Docker 快速搭建多节点 Hadoop 集群](http://dockone.io/article/395)
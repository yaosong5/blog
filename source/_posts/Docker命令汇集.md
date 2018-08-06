---
title:  Docker常用命令汇集
date: 2018年08月06日 22时15分52秒
tags:  [Dokcer]
categories: Docker
toc: true
typora-copy-images-to: ipic
---



[TOC]



# Docker源配置

  安装过程中需要重国外 docker 仓库下载文件，速度太慢，建议配置 docker 国内镜像仓库：
  vi /etc/docker/daemon.json
  {"registry-mirrors":["http://c1f0a193.m.daocloud.io"] }

启动容器

	docker run -itd  --net=br  --name slave02 --hostname slave02 centos:hadoop-spark &> /dev/null
	如果以 /bin/bash启动的话，sshd服务不会启动(docker未知bug)
## 创建容器

	--name    --hostname (同-h)  --net=    -d表示后台启动
	此命令不会打印出容器id
	docker run -itd  --net=br  --name hm --hostname hadoop-master kiwenlau/hadoop:1.0 &> /dev/null   （hadoop镜像）
	设置静态固定ip
	docker run -d --net=br --name=c6 --ip=192.168.33.6 nginx


    自动分配Ip
    docker run -d --net=br --name=c1 nginx
    
    设置docker默认ip段命令
    docker run -itd   -P -p 50070:50070 -p 8088:8088 -p 8080:8080 --name master -h master --add-host slave01:172.17.0.3 --add-host slave02:172.17.0.4 centos:ssh-spark-hadoop

## 容器挂载目录

compose文件：
```Bash
volumes:
	- /Users/yaosong/Yao/dev/hadoop/dfs/name:/root/hadoop/dfs/name
```

shell命令：

```bash
-v : 
docker run -it -v /test:/soft centos /bin/bash
```

":"前目录为宿主机目录，后目录为容器目录



## 删除所有未用的 Data volumes

```
docker volume prune
```



## run 命令解释

	-d 是后台启动
	docker run -itd  --net=br  --name spark --hostname spark yaosong/spark:2.1.0 &> /dev/null
	sudo docker exec -it hm bash（进入后台启动的容器）
	
	和下面一样（直接进入）
	docker run -it --net=br  --name spark --hostname spark yaosong/spark:2.1.0 bash

## exec 进入后台容器
	docker exec -it spark bash
	docker exec -it 容器名 bash
	执行命令 docker exec -it 容器名 ip addr 可以拿到 a0 容器的 ip

## logs查看容器启动日志

```docker logs -f -t --tail 100  kanbigdata_namenode_1```





## 查看容器信息

```
docker inspect hm
执行命令 docker exec -it 容器名 ip addr 可以拿到 a0 容器的 ip
```



## 启动 关闭 删除容器

	docker start 
	docker stop 容器名
	docker rm 容器名

## cp容器宿主互拷文件

```
docker cp /Users/yaosong/Yao/etc.tar  f7e795c0fddd:/
后为容器id:/目录
```

## 删除镜像

	（根据镜像id删除）
	docker rmi 00de07ebadff
	用docker images -a 查看image id，
	也可docker rmi 镜像名:版本号


## 保存镜像
```
docker commit -m "centos-6.9 with spark 2.2.0 and hadoop 2.8.0"  os   centos:hadoop-spark

docker commit -m "bigdata:spark,hadoop,hive,mysql and shell  foundation"   --author="yaosong"  master   yao/os/bigdata:2.1
```


## Docker用DockerFile创建镜像

	docker build -t hadoop:v1- <Dockerfile
	docker build -t="hadoop:v1" .  （.表示是当前文件夹，也就是dockerfile所在文件夹）
	docker build -f Dockerfile -t hadoop:v1 . 此命令也可

## 一键启动docker-compose.yml编排的所有服务

```
docker-compose -f docker-compose.yml up d
```

## Docker改变标签

docker tag IMAGEID(镜像id) REPOSITORY:TAG（仓库：标签）

`docker tag  b7a66cb0e8ba yaosong5/bigdata:1.0`

## 搜索docker镜像

```
docker search yaosong5
```



## 登录docker账户

`docker login` 登录docker hub中注册的账户




## 上传仓库

`docker push yaosong5/elk:1.0`

## 容器保存为镜像，加载本地镜像 引用

	docker save imageID > filename
	docker load <filename
	如：
	docker save 4f9e92e56941>  /Users/yaosong/centosSparkHadoop.tar
	docker load </Users/yaosong/centosSparkHadoop.tar
	
	通过 image 保存的镜像会保存操作历史，可以回滚到历史版本。

## 保存，加载容器命令：
	docker export containID > filename
	docker import filename [newname]
通过容器保存的镜像不会保存操作历史，所以文件小一点。
如果要运行通过容器加载的镜像， 需要在运行的时候加上相关命令。

#  Docker-machine命令
## 列出docker-machine

```
docker-machine ls
```

## 开启虚拟机

```
docker-machine start default
```

## 关闭虚拟机

```
docker-machine stop default
```

## 重启虚拟机

```
docker-machine restart default
```

## 删除虚拟机

```
docker-machine rm default
```

## 设置环境变量docker-machine

```
eval $(docker-machine env default) # Setup the environment
```




---
title:  Docker常用命令汇集
date: 2018年08月06日 22时15分52秒
tags:  [Docker]
categories: Docker
toc: true

---

![](http://img.gangtieguo.cn/006tNbRwgy1fu55c5h3g1j319i0eu0tg.jpg)

[TOC]



# Docker源配置

  安装过程中需要重国外 docker 仓库下载文件，速度太慢，建议配置 docker 国内镜像仓库：
  **vi /etc/docker/daemon.json**

```Json
  {"registry-mirrors":["http://c1f0a193.m.daocloud.io"] }
```

# 常用命令

启动容器(也是创建)

```shell
docker run -itd  --net=br  --name master --hostname master yaosong5/bigdata:1.0 &> /dev/null
如果以 /bin/bash启动的话，sshd服务不会启动(docker未知bug)
用 &> /dev/null这种方式sshd服务才会启动
```
## 创建容器-run

```shell
--name    --hostname (同-h)  --net=    -d表示后台启动
此命令不会打印出容器id
docker run -itd  --net=br  --name hm --hostname hadoop-master kiwenlau/hadoop:1.0 &> /dev/null   （hadoop镜像）
设置静态固定ip
docker run -d --net=br --name=c6 --ip=192.168.33.6 nginx
```


```shell
自动分配Ip
docker run -d --net=br --name=c1 nginx

设置docker默认ip段命令
docker run -itd   -P -p 50070:50070 -p 8088:8088 -p 8080:8080 --name master -h master --add-host slave01:172.17.0.3 --add-host slave02:172.17.0.4 centos:ssh-spark-hadoop
```

## 创建容器-Dockerfile

在dockerfille文件所在的目录中执行以下命令

```Shell
docker build -f Dockerfile -t hadoop:v1 .
#也可以指定Dockerfile如下：
docker build -t centos:tt - < Dockerfile
```



## 容器挂载目录

分两种类型

- compose文件：

```Bash
volumes:
	- /Users/yaosong/Yao/dev/hadoop/dfs/name:/root/hadoop/dfs/name
```

- shell命令：

```bash
-v : 
docker run -it -v /test:/soft centos /bin/bash
```

":"前目录为宿主机目录，后目录为容器目录



> 引用  [基于 docker 的大数据架构](https://www.kancloud.cn/huangzhenyou/shuoming/545497)
>
> 如果是mac或者win机器，需要在virtualbox虚拟机中设置共享文件夹的share名称对应mac的目录
> 虚拟机中的目录
>
> ![](http://img.gangtieguo.cn/006tKfTcgy1g0ryyzp0vrj30zi0bm0t9.jpg)
>
> sudo mount -t vboxsf vagrant /Users/yaosong
>
> sudo mount -t vboxsf Yao /Users/yaosong/Yao/
>
> 取消挂载
>
> sudo umount vagrant

## 删除所有未用的 Data volumes

```
docker volume prune
```

## run 命令解释

```Bash
-d 是后台启动
docker run -itd  --net=br  --name spark --hostname spark yaosong5/spark:2.1.0 &> /dev/null
sudo docker exec -it spark bash（进入后台启动的容器）
和下面一样（直接进入）
docker run -it --net=br  --name spark --hostname spark yaosong5/spark:2.1.0 bash
```

## pause暂停恢复容器

```sql
docker pause 容器名
恢复数据库容器db01提供服务。
docker unpause 容器名
```



## exec 进入后台容器

```Bash
docker exec -it spark bash
docker exec -it 容器名 bash
执行命令 docker exec -it 容器名 ip addr 可以拿到 a0 容器的 ip
```

## logs查看容器启动日志

```Bash
docker logs -f -t --tail 100  kanbigdata_namenode_1
```



## 查看容器信息

```shell
docker inspect hm
执行命令 docker exec -it a0 ip addr 可以拿到 a0 容器的 ip
```



## 启动 关闭 删除容器

```shell
docker start 
docker stop 容器名
docker rm 容器名
```

## cp容器宿主互拷文件

```shell
docker cp /Users/yaosong/Yao/etc.tar  f7e795c0fddd:/
后为容器id:/目录
```

## 删除镜像

```shell
（根据镜像id删除）
docker rmi 00de07ebadff
用docker images -a 查看image id，
也可docker rmi 镜像名:版本号
```


## 保存镜像
```shell
docker commit -m "centos-6.9 with spark 2.2.0 and hadoop 2.8.0"  os   centos:hadoop-spark

docker commit -m "bigdata:spark,hadoop,hive,mysql and shell  foundation"   --author="yaosong"  master   yao/os/bigdata:2.1
```


## Docker用DockerFile创建镜像

```shell
docker build -t hadoop:v1- <Dockerfile
docker build -t="hadoop:v1" .  （.表示是当前文件夹，也就是dockerfile所在文件夹）
docker build -f Dockerfile -t hadoop:v1 . 此命令也可
```

## 一键启动docker-compose.yml编排的所有服务

```shell
docker-compose -f docker-compose.yml up -d
```

## Docker改变标签

docker tag IMAGEID(镜像id) REPOSITORY:TAG（仓库：标签）

```shell
docker tag  b7a66cb0e8ba yaosong5/bigdata:1.0
```



## 搜索docker镜像

```shell
docker search yaosong5
```



## 登录docker账户

```shell
docker login
```

登录docker hub中注册的账户




## 上传仓库

```shell
docker push yaosong5/elk:1.0
```



## 容器保存为镜像，加载本地镜像 引用

```shell
docker save imageID > filename
docker load <filename
如：
docker save 4f9e92e56941>  /Users/yaosong/centosSparkHadoop.tar
docker load </Users/yaosong/centosSparkHadoop.tar

通过 image 保存的镜像会保存操作历史，可以回滚到历史版本。
```

## 保存，加载容器命令：
```powershell
docker export containID > filename
docker import filename [newname]
```
通过容器保存的镜像不会保存操作历史，所以文件小一点。
如果要运行通过容器加载的镜像， 需要在运行的时候加上相关命令。





# 高阶命令

参考：[清理Docker占用的磁盘空间](https://kiwenlau.com/2018/01/10/how-to-clean-docker-disk/)



- 删除所有关闭的容器

```shell
docker ps -a | grep Exit | cut -d ' ' -f 1 | xargs docker rm
```

- 删除所有dangling镜像(即无tag的镜像)：

```shell
docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
```

- 删除所有dangling数据卷(即无用的volume)：

```shell
docker volume rm $(docker volume ls -qf dangling=true)
```



## docker system 

它可以用于管理磁盘空间

- 磁盘使用情况

  docker system df 
  命令，类似于Linux上的df命令，用于查看Docker的磁盘使用情况

- 清理磁盘

  docker system prune   命令可以用于清理磁盘，删除关闭的容器、无用的数据卷和网络，以及dangling镜像(即无tag的镜像)。docker system prune -a命令清理得更加彻底，可以将没有容器使用Docker镜像都删掉。注意，这两个命令会把你暂时关闭的容器，以及暂时没有用到的Docker镜像都删掉了…所以使用之前一定要想清楚吶。





#  Docker-machine命令

## 列出docker-machine

```shell
docker-machine ls
```

## 开启虚拟机

```shell
docker-machine start default
```

## 关闭虚拟机

```shell
docker-machine stop default
```

## 重启虚拟机

```bash
docker-machine restart default
```

## 删除虚拟机

```bash
docker-machine rm default
```

# 设置环境变量docker-machine

```bash
eval $(docker-machine env default) # Setup the environment
```



# 其他参考

ip -4 addr

chkconfig sshd on

rpm -qa|grep ssh。

为了快速实现我们就不自己装 SSH 服务了，hub.docker.com 上的 kinogmt/centos-ssh:6.7 这个镜像就能满足我们的要求


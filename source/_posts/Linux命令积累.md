---
title:  linux命令
date: 2018年08月11日 02时55分44秒
tags:  [linux,开发]
categories: Linux
toc: true
---


简单linux命令

`nohup   & ` 
后台运行

<!-- more -->

## 文件查找
`find / -type f -size +10G`
在Linux下如何让文件让按大小单位为M,G等易读格式，S size大小排序。  `ls -lhS`
`du -h * | sort -n `  
当然您也可以结合管道文件夹内最大的几个文件  ` du -h * | sort -n|head`
动态显示机器各端口的链接情况`while :; do netstat -apn | grep ":80" | wc -l; sleep 1; done`
## sed
更改第一行 `sed -i '1s/.*//'`     sed -i '1s/.*/想更改的内容/'

```bash
ssh root@slave01 "sed -i '6c advertised.host.name=slave01 ' $KAFKA_HOME/config/server.properties"
```

删除第一行`sed -i '1d'  `     sed -i '1d' 文件名
插入第一行 `sed -i '1i\' `       sed -i ‘1i\内容‘ 文件名

## cpu
`cat /proc/cpuinfo | grep processor | wc -l`
`lscpu`

## sz rz与服务器交互上传下载文件

`sudo yum install lrzsz -y`

## 挂载

 `sshfs  root@192.168.73.12:/home/ /csdn/win10/`

即：sshfs 用户名@远程主机IP:远程主机路径 本地挂载点
sshfs  root@master:/usr/hadoop  /usr/hive/hadoop

## 查看端口是否被监听

也可验证对应端口程序是否启动

```
netstat -nl|grep 10000
netstat -antp| grep 1000
```



# tree

```shell
yum install -y tree
```

tree 可以查看目录结构

# 虚拟机共享文件夹

在virtualbox中设置共享文件夹的share名称对应mac的目录
虚拟机中的目录
sudo mount -t vboxsf vagrant /Users/yaosong

## centos查看版本

 cat /etc/redhat-release



# GLIBC2.14 not found

[libc.so.6: version 'GLIBC_2.14' not found报错提示的解决方案](https://www.cnblogs.com/kevingrace/p/8744417.html)

[【工作】Centos6.5 升级glibc解决“libc.so.6: version GLIBC_2.14 not found”报错问题](http://www.jiagoumi.com/work/811.html)

参考：[Docker-PostgresSQL](https://www.cnblogs.com/zhzhlong/p/9493245.html)

# 其他

ip -4 addr 查看ip

chkconfig sshd on

rpm -qa|grep ssh。

为了快速实现我们就不自己装 SSH 服务了，hub.docker.com 上的 kinogmt/centos-ssh:6.7 这个镜像就能满足我们的要求


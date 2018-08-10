---
title:  linux命令积累
tags:  [linux,开发]
categories: Linux
toc: true
grammar_cjkRuby: true
---

简单linux命令

`nohup   & `后台运行

<!-- more -->

## 文件查找
`find / -type f -size +10G`
在Linux下如何让文件让按大小单位为M,G等易读格式，S size大小排序。  `ls -lhS`
`du -h * | sort -n `  
当然您也可以结合管道文件夹内最大的几个文件  ` du -h * | sort -n|head`
动态显示机器各端口的链接情况`while :; do netstat -apn | grep ":80" | wc -l; sleep 1; done`
## sed
更改第一行 `sed -i '1s/.*//'`     sed -i '1s/.*/想更改的内容/'
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

`netstat -nl|grep 10000`
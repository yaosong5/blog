---
title:  ELK容器的搭建
date: 2018年08月06日 22时15分52秒
tags:  [ELK,Dokcer]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---

[TOC]



## 来源容器  elk

新建容器，为减少工作量，引用的是有ssh服务的Docker镜像**kinogmt/centos-ssh:6.7**，生成容器os为基准。

```
docker run -itd  --name elk --hostname elk kinogmt/centos-ssh:6.7 &> /dev/null
```

> 注意必须要以-d方式启动，不然sshd服务不会启动，这算是一个小bug

<!--more-->

在容器中下载需要的elk的源包。做解压就不赘述，很多案例教程。

> 我是采用的下载到宿主机，解压后，用 "docker cp 解压包目录  os:/usr/loca/"来传到容器内，比在容器内下载速度更快




## 设置Home
 `vim   ~/bashrc`

```bash
export ES_HOME=/usr/es
export PATH=$ES_HOME/bin:$PATH
export KIBANA_HOME=/usr/kibana
export PATH=$KIBANA_HOME/bin:$PATH
export LOGSTASH_HOME=/usr/logstash
export PATH=$LOGSTASH_HOME/bin:$PATH
export NODE_HOME=/usr/node
export PATH=$NODE_HOME/bin:$PATH
export NODE_PATH=$NODE_HOME/lib/node_modules
```
`source ~/.bashrc`

## 安装插件header

### 安装nodejs

一般预装的版本不对

```
yum erase nodejs npm -y   # 卸载旧版本的nodejs
rpm -qa 'node|npm' | grep -v nodesource # 确认nodejs是否卸载干净
yum install nodejs -y # 安装npm 安装的版本会有不对
```

下载合适版本

```bash
cd /usr
wget https://npm.taobao.org/mirrors/node/latest-v4.x/node-v4.4.7-llinux-x64.tar.gz
tar -zxvf node-v4.4.7-linux-x64.tar.gz
mv node-v8.9.1-linux-x64 node
```
直接将node目录配置到home即可

```
export NODE_HOME=/usr/node
export PATH=$NODE_HOME/bin:$PATH 
```



### 下载 header，安装grunt

（所有命令在hear的所在目录执行）

`wget https://github.com/mobz/elasticsearch-head/archive/master.zip `

`unzip master.zip`

看当前 head 插件目录下有无 node_modules/grunt 目录： 
没有：执行命令创建：

```
npm install grunt --save
```

安装 grunt： 
grunt 是基于 Node.js 的项目构建工具，可以进行打包压缩、测试、执行等等的工作，head 插件就是通过 grunt 启动

```
npm install -g grunt-cli
```

参考https://blog.csdn.net/ggwxk1990/article/details/78698648

 npm install 安装所下载的header包 

```
npm install
```



## header启动

在 elasticsearch-head-master 目录下

```
grunt server  或者 npm run start
```



## els不能通过root启动，创建用户

	useradd elk
	groupadd elk
	usermod -a -G elk elk
	
	echo elk | passwd --stdin elk

将elk添加到sudoers

	echo "elk ALL = (root) NOPASSWD:ALL" | tee /etc/sudoers.d/elk
	chmod 0440 /etc/sudoers.d/elk

解决sudo: sorry, you must have a tty to run sudo问题，在/etc/sudoer注释掉 Default requiretty 一行
	sudo sed -i 's/Defaults requiretty/Defaults:elk !requiretty/' /etc/sudoers



### 修改文件所有者
`chown -R elk:elk /usr/es/`

**设置资源参数**

```
  sudo vim  /etc/security/limits.d/90-nproc.conf
```


**添加**
	 elk        soft    nproc     4096
   <!-- 更改docker-machine的资源 -->
```
docker-machine ssh
sysctl -w vm.max_map_count=655360
```

### es启动脚本

    单机 su elk -c "$ES_HOME/bin/elasticsearch -d"
    ssh elk@elk1 " $ES_HOME/bin/elasticsearch -d"
    ssh root@elk1 " su elk -c  $ES_HOME/bin/elasticsearch "



### 集群
#### elasticSearch
脚本 `vim  es-start.sh`

```bash
#!/bin/bash
sed -i '6c node.name: es1 '
$ES_HOME/config/elasticsearch.yml
su - elk -c  "$ES_HOME/bin/elasticsearch -d"
ssh root@elk2 "sed -i '6c node.name: es2 ' $ES_HOME/config/elasticsearch.yml"
ssh root@elk2 ' su - elk -c  "$ES_HOME/bin/elasticsearch -d" '
ssh root@elk3 "sed -i '6c node.name: es3 ' $ES_HOME/config/elasticsearch.yml"
ssh root@elk3 ' su - elk -c  "$ES_HOME/bin/elasticsearch -d" '
```
#### kibana
启动单机（只需要启动单机） `bin/kibana`
```bash
#!/bin/bash
sed -i '3c http://elk1:9200 '
$KIBANA_HOME/config/kibana.yml
nohup $KIBANA_HOME/bin/kibana  &
ssh root@elk2 "sed -i '3c http://elk2:9200 ' $KIBANA_HOME/config/kibana.yml"
ssh root@elk2 "nohup $KIBANA_HOME/bin/kibana & "
ssh root@elk3 "sed -i '3c   http://elk3:9200 ' $KIBANA_HOME/config/kibana.yml"
ssh root@elk3 "nohup $KIBANA_HOME/bin/kibana & "
```
#### logstash
单机启动   `$LOGSTASH_HOME/bin/logstash -f logstash.conf`
 `$LOGSTASH_HOME/bin/logstash -f 配置文件的目录`


集群启动脚本 **logstash-start.sh**
```shell
#!/bin/bash
nohup $LOGSTASH_HOME/bin/logstash -f  $LOGSTASH_HOME/conf/$1  &
ssh root@elk2 "nohup $LOGSTASH_HOME/bin/logstash -f  $LOGSTASH_HOME/conf/$1 & "
ssh root@elk3 "nohup $LOGSTASH_HOME/bin/logstash -f  $LOGSTASH_HOME/conf/$1 & "
```



## 保存容器为镜像

```bash
docker commit -m "elk镜像"  --author="yaosong"  os  yaosong5/elk:1.0
```



## 生成elk 容器

```shell
docker run -itd --net=br --name elk1 --hostname elk1 yaosong5/elk:1.0 &> /dev/null
docker run -itd --net=br --name elk2 --hostname elk2 yaosong5/elk:1.0 &> /dev/null
docker run -itd --net=br --name elk3 --hostname elk3 yaosong5/elk:1.0 &> /dev/null
```



### 停止/删除elk 容器

```shell
docker stop elk1
docker stop elk2
docker stop elk3

docker rm elk1
docker rm elk2
docker rm elk3
```
## 参考
## elk 操作命令
### es操作命令
http://www.yfshare.vip/2017/11/04/%E9%83%A8%E7%BD%B2FileBeat-logstash-elasticsearch%E9%9B%86%E7%BE%A4-kibana/#%E9%85%8D%E7%BD%AE-filebeart



## 其他
yum erase nodejs npm -y   # 卸载旧版本的nodejs
rpm -qa 'node|npm' | grep -v nodesource # 确认nodejs是否卸载干净
yum install nodejs -y

---
title:  Docker-machine的创建，mac宿主机和docker容器网络互通Docker容器与宿主机在同一ip段下
date: 2018年07月20日 03时02分26秒
tags:  [Docker,Docker-machine,安装部署]
categories: Docker
toc: true
---



![enter description here](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/7/20/1532027033015.jpg)

此文纯属命令记录，后续更新原理解说

<!-- more -->

 # 更改virtual0的ip
 VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.33.253 --netmask 255.255.255.0

# ifconfig 查看
创建虚拟机配置文件  Vagrantfile

也可以vagrant init  会生成一个空白的Vagrantfile
# vi Vagrantfile

```SHELL
	Vagrant.configure(2) do |config|
		 config.vm.box = "dolbager/centos-7-docker"
		 config.vm.hostname = "default"
		  config.vm.network "private_network", ip: "192.168.33.1",netmask: "255.255.255.0"
		 config.vm.provider "virtualbox" do |v|
		   v.name = "default"
		   v.memory = "2048"
		   # Change the network adapter type and promiscuous mode
		   v.customize ['modifyvm', :id, '--nictype1', 'Am79C973']
		   v.customize ['modifyvm', :id, '--nicpromisc1', 'allow-all']
		   v.customize ['modifyvm', :id, '--nictype2', 'Am79C973']
		   v.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
		 end
		 # Install bridge-utils
		 config.vm.provision "shell", inline: <<-SHELL
		    curl -o /etc/yum.repos.d/CentOS-Base.repohttp://mirrors.aliyun.com/repo/Centos-7.repo
		    curl -o /etc/yum.repos.d/epel.repohttp://mirrors.aliyun.com/repo/epel-7.repo
		   yum clean all
		   yum makecache
		   yum update -y
		   yum install bridge-utils net-tools -y
		 SHELL
	end

```
vagrant up
vagrant ssh

vagrant ssh-config

```bash
scp ~/.vagrant.d/boxes/dolbager-VAGRANTSLASH-centos-7-docker/0.2/virtualbox/vagrant_private_key .vagrant/machines/default/virtualbox/private_key

```

`vagrant exit`






```bash
docker-machine create \
 --driver "generic" \
 --generic-ip-address 192.168.33.1 \
 --generic-ssh-user vagrant \
 --generic-ssh-key .vagrant/machines/default/virtualbox/private_key \
 --generic-ssh-port 22 \
 default
```

创建网桥docker1 和 docker network br
通过vagrant 从虚拟机的 eth0 登录到虚拟机

vagrant ssh
ip -4 addr

创建 docker network br


```bash
sudo docker network create \
    --driver bridge \
    --subnet=192.168.33.0/24 \
    --gateway=192.168.33.1 \
    --opt "com.docker.network.bridge.enable_icc"="true" \
    --opt "com.docker.network.bridge.enable_ip_masquerade"="true" \
    --opt "com.docker.network.bridge.name"="docker1" \
    --opt "com.docker.network.driver.mtu"="1500" \
    br

```


创建网桥配置文件docker1
![A82E4EE8-9517-4B66-A6CB-FE3C964B6D99](http://img.gangtieguo.cn/006tKfTcgy1ftom0jyp9vj30kl07l76c.jpg)

`vim /etc/sysconfig/network-scripts/ifcfg-docker1`

```bash
DEVICE=docker1
TYPE=Bridge
BOOTPROTO=static
ONBOOT=yes
STP=on
IPADDR=
NETMASK=
GATEWAY=
DNS1=
```


## 修改网卡配置 eth1 :

`sudo vi /etc/sysconfig/network-scripts/ifcfg-eth1`

```bash
DEVICE=eth1
BOOTPROTO=static
HWADDR=
ONBOOT=yes
NETMASK=
GATEWAY=
BRIDGE=docker1
TYPE=Ethernet
```

`ip -4 addr`

![E77B4516-B852-4E7F-A876-FCF01FDB93BC](http://img.gangtieguo.cn/006tKfTcgy1ftom2nhzd3j31ey0luqhe.jpg)







初始化docker-machine变量

提示报错Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running? 

```bash
 eval $(docker-machine env default) 
 如果要取消变量 
  eval $(docker-machine env -u)
 
```



参考：[docker-install-mac-vm-centos](https://github.com/SixQuant/engineering-excellence/blob/master/docker/docker-install-mac-vm-centos.md)


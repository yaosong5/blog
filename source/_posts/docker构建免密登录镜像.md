---
title:  Docker构建免密ssh镜像
date: 2018年06月21日 22时15分52秒
tags:  [Docker]
categories: Docker
toc: true
---

##  免密登录
参考的是http://www.shushilvshe.com/data/docker-ssh.html
文中涉及命令
```
sudo yum -y install openssh-server openssh-clients
ssh-keygen -t rsa
cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

<!--more-->
**vim /etc/ssh/sshd_config**
　　找到以下内容，并去掉注释符”#“

```
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
```

**vim /etc/ssh/ssh_config**

```
Host *
	StrictHostKeyChecking no
	UserKnownHostsFile=/dev/null
```

​		此文也可参考 http://www.voidcn.com/article/p-gxkeusey-ma.html



> https://blog.csdn.net/a85820069/article/details/78745899
> 坑
> 使用 docker run -i -t –name c1 centos6.6:basic /bin/bash 运行容器，sshd 服务是不开启的，必须先 - d 在用 exec 切入。


https://www.cnblogs.com/aiweixiao/p/5516974.html


1.【查看是否启动】

　　启动 SSH 服务 “/etc/init.d/sshd start”。然后用 netstat -antulp | grep ssh 看是否能看到相关信息就可以了。

2.【设置自动启动】

　　如何设置把 ssh 等一些服务随系统开机自动启动？

		方法一：[root@localhost ~]# vi /etc/rc.local
加入：service sshd start 或  /etc/init.d/sshd start



**chmod 777 /etc/ssh/ssh_host_ecdsa_key**

```
# 免密登录

mkdir -p /root/.ssh  
touch /root/.ssh/config  
echo "StrictHostKeyChecking no" > /root/.ssh/config  
sed -i "a UserKnownHostsFile /dev/null" /root/.ssh/config 

# 开机启动

RUN yum install -y openssh-server sudo  
RUN sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config  

# 下面这两句比较特殊，在centos6上必须要有，否则创建出来的容器sshd不能登录

RUN ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key  
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key 
```



## ssh服务文章当中的
```bash
curl http://mirrors.aliyun.com/repo/Centos-6.repo > /etc/yum.repos.d/CentOS-Base-6-aliyun.repo
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
yum makecache
yum install -y net-tools which openssh-clients openssh-server iproute.x86_64 wget

service sshd start

sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config
sed -ri 's/session required pam_loginuid.so/#session required pam_loginuid.so/g' /etc/pam.d/sshd

chkconfig sshd on

cd ~;ssh-keygen -t rsa -P '' -f ~/.ssh/id_dsa;cd .ssh;cat id_dsa.pub >> authorized_keys
```

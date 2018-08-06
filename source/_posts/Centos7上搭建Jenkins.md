---
title:  Centos7上搭建Jenkins
date: 2018年06月21日 22时15分52秒
tags:  [Jenkins]
categories: 安装部署
toc: true
---
![Jenkins](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/21/1529592059717.jpg)



之前用yum模式安装，总是启动报错，解决了一番，未找到解决方案，后直接下载war包进行安装部署

默认安装了Java
<!-- more -->

# 1. 安装 jenkins

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=480580003&auto=1&height=66"></iframe>

``` bash
cd /opt
mkdir /jenkins
cd jenkins
mkdir jenkins_home
mkdir jenkins_node
wget http://mirrors.jenkins-ci.org/war/latest/jenkins.war
```


# 2. 编写可执行文件

  ` vim start_jenkins.sh`
```bash
     #!/bin/bash
     JENKINS_ROOT=/opt/jenkins
     export JENKINS_HOME=$JENKINS_ROOT/jenkins_home
     java -jar $JENKINS_ROOT/jenkins.war --httpPort=8000
```
   修改文件的权限： ` chmod  a+x   start_jenkins.sh`

   启动 jenkins:  `   nohup ./start_jenkins.sh > jenkins.log 2>& 1& `               
# 3 访问 jenkins
   输入 http:// 服务器地址: 8000

注意：在启动日志中会出现初始密码，这个用来首次登陆Jenkins使用
![初始密码](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/21/1529590960879.jpg)


![初次登陆的界面](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/6/21/1529591748025.jpg)

参考
[在 Centos7 上搭建 jenkins](https://blog.csdn.net/python_tty/article/details/52884314)
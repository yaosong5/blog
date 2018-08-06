---
title:  Linux安装mysql
date: 2018年06月21日 22时15分52秒
tags:  [Linux,mysql]
categories: 安装
toc: true
---

```bash
yum install -y mysql-server
chkconfig --add mysqld
chkconfig mysqld on
chkconfig --list mysqld
service mysqld start
mysql -u root -p
Enter password:           //默认密码为空，输入后回车即可
set password for root@localhost=password('root'); 　　密码设置为root
set password for root@=password('root');
默认情况下Mysql只允许本地登录，所以只需配置root@localhost就好
设置所有ip访问密码为root
set password for root@%=password('root'); 　　　　　　密码设置为root （其实这一步可以不配）
设置master访问密码为root
set password for root@master=password('root'); 　　密码设置为root （其实这一步可以不配）
查询密码
select user,host,password from mysql.user;  　　查看密码是否设置成功
设置所有ip可以通过root访问
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%' IDENTIFIED BY 'hive' WITH GRANT OPTION;

```


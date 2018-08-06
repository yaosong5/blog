docker run -itd  --net=br  --name cm --hostname cm yaosong5/centosbase:1.0 &> /dev/null




将下载的包进行解压然后进行拷贝

docker cp /Users/yaosong/Yao/cloudera-manager-el6-cm5.9.0_x86_64/cloudera 8a3b1b4b5386:/opt/
docker cp /Users/yaosong/Yao/cloudera-manager-el6-cm5.9.0_x86_64/cm-5.9.0 8a3b1b4b5386:/opt/

docker cp /Users/yaosong/Yao/mysql-connector-java-5.1.40-bin.jar 8a3b1b4b5386:/opt/cm-5.9.0/share/cmf/lib/

docker cp /Users/yaosong/Yao/mysql-connector-java.jar 8a3b1b4b5386:/usr/share/java/

docker cp /Users/yaosong/Yao/jdk1.8 8a3b1b4b5386:/usr/local/




将 parcel 文件放至 /opt/cloudera/parcel-repo

docker cp /Users/yaosong/Yao/CDH-5.9.0-1.cdh5.9.0.p0.23-el6.parcel  8a3b1b4b5386:/opt/cloudera/parcel-repo
docker cp /Users/yaosong/Yao/CDH-5.9.0-1.cdh5.9.0.p0.23-el6.parcel.sha 8a3b1b4b5386:/opt/cloudera/parcel-repo
docker cp /Users/yaosong/Yao/manifest.json 8a3b1b4b5386:/opt/cloudera/parcel-repo




vim /etc/profile
export JAVA_HOME=/usr/local/jdk1.8

export PATH=$JAVA_HOME/bin:$PATH

export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar


usr/share/java 并重新命名为   mysql-connector-java.jar

初始化 mysql 库

[root@server1 cm-5.9.0]# /opt/cm-5.9.0/share/cmf/schema/scm_prepare_database.sh mysql cm -hlocalhost -uroot -proot --scm-host localhost scm scm scm


创建用户（所有节点执行）

[root@server1 cm-5.9.0]# useradd --system --home=/opt/cm-5.9.0/run/cloudera-scm-server/ --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm


Agent 配置 

vim /opt/cm-5.9.0/etc/cloudera-scm-agent/config.ini

 将 server_host 改为主节点主机名

server_host=cm1




安装mysql
chkconfig mysqld on

5、设置允许远程登录

mysql -u root -p 
你的密码
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION; 
6、创建CM用的数据库

安装集群时按需创建，详见第七章第13步

--hive数据库
create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
--oozie数据库
create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
--hue数据库
create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;





Cloudera推荐设置

在试安装的过程，发现Cloudera给出了一些警告，如下图：



身为一个有洁癖的码农，自然是连黄色的感叹号都要消灭的。因此在安装CM/CDH之前就先全部设置好。

1、设置swap空间

vim /etc/sysctl.conf
末尾加上
vm.swappiness=10

2、关闭大页面压缩

试过只设置defrag，但貌似个别节点还是会有警告，干脆全部设置

vim /etc/rc.local
末尾加上(永久生效)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag


创建 cdh 所需要的库

create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;

create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;

create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;

create database monitor DEFAULT CHARSET utf8 COLLATE utf8_general_ci;

create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;












89dde192b411:/cdh-rpm

docker cp /Users/yaosong/Yao/cloudera-manager-agent-5.9.0-1.cm590.p0.249.el6.x86_64.rpm 89dde192b411:/cdh-rpm
docker cp /Users/yaosong/Yao/cloudera-manager-daemons-5.9.0-1.cm590.p0.249.el6.x86_64.rpm 89dde192b411:/cdh-rpm
docker cp /Users/yaosong/Yao/cloudera-manager-server-5.9.0-1.cm590.p0.249.el6.x86_64.rpm 89dde192b411:/cdh-rpm
docker cp /Users/yaosong/Yao/cloudera-manager-server-db-2-5.9.0-1.cm590.p0.249.el6.x86_64.rpm 89dde192b411:/cdh-rpm
docker cp /Users/yaosong/Yao/enterprise-debuginfo-5.9.0-1.cm590.p0.249.el6.x86_64.rpm 89dde192b411:/cdh-rpm
docker cp /Users/yaosong/Yao/jdk-6u31-linux-amd64.rpm 89dde192b411:/cdh-rpm
docker cp /Users/yaosong/Yao/oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm  89dde192b411:/cdh-rpm







启动cloudera manager 服务
/opt/cm-5.9.0/etc/init.d/cloudera-scm-server start

/opt/cm-5.9.0/etc/init.d/cloudera-scm-agent start



端口 7180



保存为镜像：
	docker commit -m "cloudera manger image"  cm   yaosong5/cm59:1.0
创建容器：
	docker run -itd  --net=br  --name cm1 --hostname cm1 yaosong5/cm59:1.0 &> /dev/null
	docker run -itd  --net=br  --name cm2 --hostname cm2 yaosong5/cm59:1.0 &> /dev/null
	docker run -itd  --net=br  --name cm3 --hostname cm3 yaosong5/cm59:1.0 &> /dev/null



docker stop cm1
docker stop cm2
docker stop cm3


docker rm cm1
docker rm cm2
docker rm cm3





调错

.SearchRepositoryManager: No read permission to the server storage directory [/var/lib/cloudera-scm-server]
2018-07-11 10:20:39,788 ERROR SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: No write permission to the server storage directory [/var/lib/cloudera-scm-server]

链接hue连接不上
节点的 cm-5.x.0/log/cloudera-scm-server/cloudera-scm-server.log，一般情况下应该会说到

ImportError:libxslt.so.1:cannot open shared object file:No such file ordirectory

yum -y install libxml2-python 

提示hue测试连接连接不上，安装依赖：

yum install libxml2-python  mod_ssl install krb5-devel cyrus-sasl-gssapi cyrus-sasl-deve libxml2-devel libxslt-devel mysql mysql-devel openldap-devel python-devel python-simplejson sqlite-devel -y


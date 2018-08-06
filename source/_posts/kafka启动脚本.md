---
title:  kafka的启动及命令
date: 2018年06月21日 22时15分52秒
tags:  [Kakfa]
categories: 大数据
toc: true
---
# kafka的启动脚本

 	kafka-startall.sh

	#!/bin/bash
	sed -e '1c borker.id=0' $KAFKA_HOME/config/server.properties
	$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties
	ssh root@slave01 "sed -i '1c borker.id=1 ' $KAFKA_HOME/config/server.properties"
	ssh root@slave01 "sed -i '5c host.name=slave01 ' $KAFKA_HOME/config/server.properties"
	ssh root@slave01 "sed -i '6c advertised.host.name=slave01 ' $KAFKA_HOME/config/server.properties"
	ssh root@slave01 "$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties"
	ssh root@slave02 "sed -i '1c borker.id=2 ' $KAFKA_HOME/config/server.properties"
	ssh root@slave02 "sed -i '5c host.name=slave02 ' $KAFKA_HOME/config/server.properties"
	ssh root@slave02 "sed -i '6c advertised.host.name=slave02 ' $KAFKA_HOME/config/server.properties"
	ssh root@slave02 "$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties"
<!--more -->

	kafka的shell命令
	/usr/kafka/bin/kafka-topics.sh --create --zookeeper hbasezk1:2181 --replication-factor 2 --partitions 1 --topic shuaige
	/usr/kafka/bin/kafka-console-producer.sh --broker-list master:9092 --topic shuaige
	
	'''在一台服务器上创建一个订阅者'''
	/usr/kafka/bin/kafka-console-consumer.sh --zookeeper hbasezk1:2181 --topic shuaige --from-beginning
	/kafka-topics.sh --list --zookeeper localhost:12181



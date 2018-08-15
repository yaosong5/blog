---
title:  kafka的启动及常用命令
date: 2018年06月21日 22时15分52秒
tags:  [Kafka]
categories: 大数据
toc: true
---
# kafka的启动脚本

 	kafka-startall.sh

```bash
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
```
<!--more -->

# kafka的shell命令

jps -ml 查看kafka的运行情况

## 启动

```bash
nohup  $KAFKA_HOME/bin/kafka-server-start.sh  $KAFKA_HOME/config/server.properties > /dev/null 2>&1 &
/usr/kafka/bin/kafka-server-start.sh -daemon /usr/kafka/config/server.properties
```

## 创建 topic

```bash
$KAFKA_HOME/bin/kafka-topics.sh --create --zookeeper zk1:2181 --replication-factor 2 --partitions 3 --topic shuaige
```

## 查看消费位置

```bash
sh  $KAFKA_HOME/bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper zk1:2181 --group testGroup

```

## 查看某个 Topic 的详情

```bash
sh  $KAFKA_HOME/bin/kafka-topics.sh --topic test --describe --zookeeper zk1:2181

```

## 创建生产者

```bash
$KAFKA_HOME/bin/kafka-console-producer.sh --broker-list kafka1:9092 --topic shuaige
```

## 创建消费者

```bash
$KAFKA_HOME/bin/kafka-console-consumer.sh --zookeeper zk1:2181 --topic shuaige --from-beginning

```

## 查看所有 topic

zk 表示 zk 的地址 地址表示 zookeeper 的地址

```Bash
$KAFKA_HOME/bin/kafka-topics.sh --list --zookeeper  zk1:2181
```

## 删除 topic

```bash
sh $KAFKA_HOME/bin/kafka-topics.sh --delete --zookeeper zk1:2181 --topic test
```
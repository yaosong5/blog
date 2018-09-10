---
title:  kafka的启动及常用命令
date: 2018年06月21日 22时15分52秒
tags:  [Kafka]
categories: 大数据
toc: true
---
# kafka的启动脚本

 kafka集群没有脚本能直接启动所有的节点，所以为了繁杂去每台机器启动，所以编写了脚本启动所有

注意参照替换



**vim  kafka-startall.sh**

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
$KAFKA_HOME/bin/kafka-topics.sh --create --zookeeper zk1:2181,zk2:2181,zk3:2181 --replication-factor 2 --partitions 3 --topic shuaige
```

## 查看消费位置

```bash
$KAFKA_HOME/bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper zk1:2181 --group testGroup
```

## 查看某个 Topic 的详情

```bash
$KAFKA_HOME/bin/kafka-topics.sh --topic test --describe --zookeeper zk1:2181,zk2:2181,zk3:2181
```

## producer创建命令行消息生产者

```bash
$KAFKA_HOME/bin/kafka-console-producer.sh --broker-list kafka1:9092,kafka2:9092,kafka3:9092 --topic shuaige
```

## Consumer创建命令行消费者

也可以说是查看message

> 注意：从 kafka-0.9 版本及以后，kafka 的消费者组和 offset 信息就不存 zookeeper 了，而是存到 broker 服务器上，所以，如果你为某个消费者指定了一个消费者组名称（group.id），那么，一旦这个消费者启动，这个消费者组名和它要消费的那个 topic 的 offset 信息就会被记录在 broker 服务器上。

### old版本

```Bash
$KAFKA_HOME/bin/kafka-console-consumer.sh --zookeeper zk1:2181,zk2:2181,zk3:2181 --topic shuaige --from-beginning  --group testGroup
```

### new版本

```Bash
$KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server kafka1:9092,kafka2:9092,kafka3:9092 --topic shuaige --from-beginning --group testGroup 
```

消费者要从头开始消费某个 topic 的全量数据，需要满足 2 个条件（spring-kafka）：

```
（1）使用一个全新的"group.id"（就是之前没有被任何消费者使用过）;
（2）指定"auto.offset.reset"参数的值为earliest；
```

对应的 spring-kafka 消费者客户端配置参数为：

```
<!-- 指定消费组名 -->
<entry key="group.id" value="fg11"/>
<!-- 从何处开始消费,latest 表示消费最新消息,earliest 表示从头开始消费,none表示抛出异常,默认latest -->
<entry key="auto.offset.reset" value="earliest"/>
```



## 查看所有 topic

zk 表示 zk 的地址 地址表示 zookeeper 的地址

```Bash
$KAFKA_HOME/bin/kafka-topics.sh --list --zookeeper  zk1:2181,zk2:2181,zk3:2181
```

## 删除 topic

```bash
sh $KAFKA_HOME/bin/kafka-topics.sh --delete --zookeeper zk1:2181,zk2:2181,zk3:2181 --topic test
```
---
title: Kafka初步总结
date: 2018年08月06日 22时15分52秒
tags: [Kafka]
categories: 总结
toc: true
---

[TOC]

>  Kafka是一个分布式消息队列：生产者、消费者的功能。它提供了类似于JMS的特性，但是在设计实现上完全不同，此外它并不是JMS规范的实现。
>  Kafka对消息保存时根据Topic进行归类，发送消息者称为Producer,消息接受者称为Consumer,此外kafka集群有多个kafka实例组成，每个实例(server)成为broker。
>  无论是kafka集群，还是producer和consumer都依赖于zookeeper集群保存一些meta信息，来保证系统可用性
>

### JMS的基础

JMS是什么：JMS是Java提供的一套技术规范

JMS干什么用：用来异构系统 集成通信，缓解系统瓶颈，提高系统的伸缩性增强系统用户体验，使得系统模块化和组件化变得可行并更加灵活

通过什么方式：生产消费者模式（生产者、服务器、消费者）

![](https://img.gangtieguo.cn/006tNbRwgy1fu895melcpj30zd0ckdgd.jpg)

jdk，kafka，activemq……

<!-- more -->

### JMS消息传输模型

点对点模式**（一对一，消费者主动拉取数据，消息收到后消息清除）**

点对点模型通常是一个基于拉取或者轮询的消息传送模型，这种模型从队列中请求信息，而不是将消息推送到客户端。这个模型的特点是发送到队列的消息被**一个且只有一个接收者接收处理**，即使有多个消息监听者也是如此。

发布/订阅模式**（一对多，数据生产后，推送给所有订阅者）**

发布订阅模型则是一个基于推送的消息传送模型。发布订阅模型可以有多种不同的订阅者，临时订阅者只在主动监听主题时才接收消息，而持久订阅者则监听主题的所有消息，**即时当前订阅者不可用，处于离线状态**。

![](https://img.gangtieguo.cn/0069RVTdgy1fu894keo4oj310n0hd0tv.jpg)

queue.put（object）  数据生产

queue.take(object)    数据消费

**kafka是采用的类jms模式，与类jms模式区别是**： 

jms两种模式：1，推送的话可以多个，2拉取的话只能一个消费者，因为消费完，消息数据就会不存在了 kafka解决了这种弊端，拉取模式下也可以多个消费者，因为消息可以持久化到硬盘，就算消费了也是存在的。Kafka中ack机制是为了保证消息完整被处理 



# kafka是什么

    类JMS消息队列，结合JMS中的两种模式，可以有多个消费者主动拉取数据，在JMS中只有点对点模式才有消费者主动拉取数据。
    kafka是一个生产-消费模型。
    Producer：生产者，只负责数据生产，生产者的代码可以集成到任务系统中。 
              数据的分发策略由producer决定，默认是defaultPartition  Utils.abs(key.hashCode) % numPartitions
    Broker：当前服务器上的Kafka进程,俗称拉皮条。只管数据存储，不管是谁生产，不管是谁消费。
            在集群中每个broker都有一个唯一brokerid，不得重复。
    Topic:目标发送的目的地，这是一个逻辑上的概念，落到磁盘上是一个partition的目录。partition的目录中有多个segment组合(index,log)
            一个Topic对应多个partition[0,1,2,3]，一个partition对应多个segment组合。一个segment有默认的大小是1G。
            每个partition可以设置多个副本(replication-factor 1),会从所有的副本中选取一个leader出来。所有读写操作都是通过leader来进行的。
            特别强调，和mysql中主从有区别，mysql做主从是为了读写分离，在kafka中读写操作都是leader。
    ConsumerGroup：数据消费者组，ConsumerGroup可以有多个，每个ConsumerGroup消费的数据都是一样的。
                   可以把多个consumer线程划分为一个组，组里面所有成员共同消费一个topic的数据，组员之间不能重复消费。
                   

## Kafka核心组件
Topic ：消息根据Topic进行归类
Producer：发送消息者
Consumer：消息接受者
broker：每个kafka实例(server)
Zookeeper：依赖集群保存meta信息。

![](https://img.gangtieguo.cn/006tNbRwgy1fu89ajoxc5j30mf0fnt9d.jpg)



## 为什么需要消息队列

消息系统的核心作用就是三点：解耦，异步和并行

以用户注册的案列来说明消息系统的作用



## 消息队列和rpc调用区别

​    消息队列并不关心是哪个消费者消费了数据，发布成功后就不必管消息队列的内容是否被消费，但是rpc调用的话，必须要给调用的系统返回一个状态码

以下为针对Kafka的一些总结


# kafka生产数据时的分组策略
    默认是defaultPartition  Utils.abs(key.hashCode) % numPartitions
    上文中的key是producer在发送数据时传入的，produer.send(KeyedMessage(topic,myPartitionKey,messageContent))

# kafka如何保证数据的完全生产
    ack机制：broker表示发来的数据已确认接收无误，表示数据已经保存到磁盘。
    0：不等待broker返回确认消息
    1：等待topic中某个partition leader保存成功的状态反馈
    -1：等待topic中某个partition 所有副本都保存成功的状态反馈
    
# broker如何保存数据
    在理论环境下，broker按照顺序读写的机制，可以每秒保存600M的数据。主要通过pagecache机制，尽可能的利用当前物理机器上的空闲内存来做缓存。
    当前topic所属的broker，必定有一个该topic的partition，partition是一个磁盘目录。partition的目录中有多个segment组合(index,log)

# partition如何分布在不同的broker上


```java
 int i = 0
    list{kafka01,kafka02,kafka03}
    
    for(int i=0;i<5;i++){
        brIndex = i%broker;
        hostName = list.get(brIndex)
    }
```


    
# consumerGroup的组员和partition之间如何做负载均衡
    最好是一一对应，一个partition对应一个consumer。
    如果consumer的数量过多，必然有空闲的consumer。
    
    算法：
        假如topic1,具有如下partitions: P0,P1,P2,P3
        加入group中,有如下consumer: C1,C2
        首先根据partition索引号对partitions排序: P0,P1,P2,P3
        根据consumer.id排序: C0,C1
        计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size,本例值M=2(向上取整)
        然后依次分配partitions: C0 = [P0,P1],C1=[P2,P3],即Ci = [P(i * M),P((i + 1) * M -1)]

# 如何保证kafka消费者消费数据是全局有序的
    如果要全局有序的，必须保证生产有序，存储有序，消费有序。
    由于生产可以做集群，存储可以分片，消费可以设置为一个consumerGroup，要保证全局有序，就需要保证每个环节都有序。
    只有一个可能，就是一个生产者，一个partition，一个消费者。这种场景和大数据应用场景相悖。
    1.生产者是集群模式--》全局序号管理器
    2.broker断只设置一个partion-》kafka的高并发下的负载均衡
    3.消费者如果是一个组，如何保障消息有序？消费来一个线程（自定义一个数据结构来排序）
    



不被完整处理，会造成结果？ 
是否开启ack-fail机制需要根据**业务场景来** 在大数据操作**点击流数据**基本上是不开启的，点击流日志中一条pv，uv数据丢失不会造成什么影响。
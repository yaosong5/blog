---
title:  ELK流程小实例
date: 2018年09月09日 
tags:  [ELK]
categories: 大数据
toc: true
---



elk流程实例



# 将日志写入到kafka



## 准备工作创建topic

先创建对应topic，（按照机器配置，集群情况配置，不然默认创建的话，不是最优效果）

<!--more-->
```
$KAFKA_HOME/bin/kafka-topics.sh --create --zookeeper zk1:2181,zk2:2181,zk3:2181 --replication-factor 2 --partitions 3 --topic gamekafka
```

![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv2f6t8yo2j31kw028q64.jpg)

查看topic是否创建成功

```
$KAFKA_HOME/bin/kafka-topics.sh --list --zookeeper  zk1:2181,zk2:2181,zk3:2181
```

![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv34y8la2jj31ha0380uq.jpg)



## 编辑写入kafka的logstash配置文件



**vim $LOGSTASH_HOME/conf/logstash-game-kafka.conf**



```
input {
  file {
      codec=>plain {
      charset => "UTF-8"
      }
      #这是日志的路径
    path => "/BaseDir/2016-02-01/*.txt"
    discover_interval => 5
    start_position => "beginning" 
  }
}

output {
    kafka {
      topic_id => "gamekafka"
      codec => plain {
        format => "%{message}"
        charset => "UTF-8"
      }
      #kafka的地址，端口为9092
      bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    }
    stdout {codec => rubydebug}
}
```

启动命令

## 启动命令

```
$LOGSTASH_HOME/bin/logstash  -f $LOGSTASH_HOME/conf/logstash-game-kafka.conf
```



启动成功

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fv2i380wmgj31kw0ig1kx.jpg)



## 查看kafka中是否有数据

```
$KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server   "kafka1:9092,kafka2:9092,kafka3:9092" --topic gamekafka --from-beginning --group testGroup
```

![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv34s3tayyj31kw02w78p.jpg)



## logstash报错

若测试报错内存不够，参考 https://blog.csdn.net/qq_23598037/article/details/79512677

在es和logstash的配置目录jvm.options中设置更小内存

```
-Xms400m  
-Xmx400m
```

# 将kafka中的数据写入到es

当然要先启动es啦（此处省略）

## 编辑配置文件

配置文件 **vim $LOGSTASH_HOME/conf/logstash-game-kafka-es.conf**

```
input {
  kafka {
    codec => "plain"
    group_id => "es2"
    bootstrap_servers => ["kafka1:9092,kafka2:9092,kafka3:9092"] # 注意这里配置的kafka的broker地址不是zk的地址
    auto_offset_reset => "earliest"
    topics => ["gamekafka"]
  }
}

filter {
 
    mutate {
      split => { "message" => "    " }
      add_field => {
        "event_type" => "%{message[3]}"
        "current_map" => "%{message[4]}"
        "current_X" => "%{message[5]}"
        "current_y" => "%{message[6]}"
        "user" => "%{message[7]}"
        "item" => "%{message[8]}"
        "item_id" => "%{message[9]}"
        "current_time" => "%{message[12]}"
     }
     remove_field => [ "message" ]
   }
}

output {
    elasticsearch {
      index => "gamelogs"
      codec => plain {
        charset => "UTF-8"
      }
      hosts => ["elk1:9200", "elk2:9200", "elk3:9200"]
    }
  stdout {codec => rubydebug}
}
```

再启动一个

```
$LOGSTASH_HOME/bin/logstash  -f $LOGSTASH_HOME/conf/logstash-game-kafka-es.conf
```



再查看es里面是否有数据 

![](https://ws1.sinaimg.cn/large/0069RVTdgy1fv34r53q6tj31kw09840a.jpg)

大功告成

# 也可以直接从logstash中读取数据到es

配置文件

**vim  logstash-game-file-es.conf** 

```
input {
  file {
      codec=>plain {
      charset => "UTF-8"
      }
    path => "/BaseDir/2016-02-01/*.txt"
    discover_interval => 5
    start_position => "beginning" 
  }
}
filter {
 
    mutate {
      split => { "message" => "    " }
      add_field => {
        "event_type" => "%{message[3]}"
        "current_map" => "%{message[4]}"
        "current_X" => "%{message[5]}"
        "current_y" => "%{message[6]}"
        "user" => "%{message[7]}"
        "item" => "%{message[8]}"
        "item_id" => "%{message[9]}"
        "current_time" => "%{message[12]}"
     }
     remove_field => [ "message" ]
   }
}

output {
    elasticsearch {
      index => "kafkagamelogs"
      codec => plain {
        charset => "UTF-8"
      }
      hosts => ["elk1:9200", "elk2:9200", "elk3:9200"]
    }
  stdout {codec => rubydebug}
}
```

启动

```
$LOGSTASH_HOME/bin/logstash  -f $LOGSTASH_HOME/conf/logstash-game-file-es.conf
```






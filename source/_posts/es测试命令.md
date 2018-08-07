---
title:  ES测试命令
date: 2018年08月06日 22时15分52秒
tags:  [ELK,Docker,es]
categories: 安装部署
toc: true
typora-copy-images-to: ipic
---

[TOC]

简单命令测试和展示es的功能

<!--more -->

插入记录

```
curl  -H "Content-Type: application/json"  -XPUT 'http://localhost:9200/store/books/1' -d '{

"title": "Elasticsearch: The Definitive Guide",
"name" : {
    "first" : "Zachary",
    "last" : "Tong"
},
"publish_date":"2015-02-06",
"price":"49.99"

}'


```

在添加一个书的信息
```
curl  -H "Content-Type: application/json"  -XPUT 'http://elk1:9200/store/books/2' -d '{
"title": "Elasticsearch Blueprints",
"name" : {
    "first" : "Vineeth",
    "last" : "Mohan"
},
"publish_date":"2015-06-06",
"price":"35.99"
}'
```

通过ID获得文档信息

```
curl   -H "Content-Type: application/json"  -XGET 'http://elk1:9200/store/books/1'
```



```
curl  -H "Content-Type: application/json"  -XGET 'http://elk1:9200/store/books/_search' -d '{
"query" : {

"filtered" : {
    "query" : {
        "match_all" : {}
        },
        "filter" : {
            "term" : {
                "price" : 35.99
            }
        }
    }
}

}'
```



在浏览

 	

```
curl -H "Content-Type: application/json"  -XPUT 'http://elk1:9200/store/books/1' -d '{

"title": "Elasticsearch: The Definitive Guide",
"name" : {
    "first" : "Zachary",
    "last" : "Tong"
},
"publish_date":"2015-02-06",
"price":"49.99"

}'
```



```
curl -H "Content-Type: application/json" -XPUT 'http://127.0.0.1:9200/kc22k2_test’ -d ‘
```



```
curl -XPUT elk1:9200/test
```



```
curl -XGET 'http://elk1:9200/_cluster/state?pretty'
{
  "error" : {

"root_cause" : [
  {
    "type" : "master_not_discovered_exception",
    "reason" : null
  }
],
"type" : "master_not_discovered_exception",
"reason" : null

  },
  "status" : 503
}
```


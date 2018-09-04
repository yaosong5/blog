---
title: SparkStreaming介绍
date: 2018年08月06日 22时15分52秒
tags: [Spark,SparkStreaming,原理]
categories: 大数据
toc: true
---

[TOC]

大数据领域，分为离线计算和实时计算 

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuaxkz7halj30i6057dfw.jpg)

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuaxl270fuj30kj034jrd.jpg)

<!-- more -->

## Streaming和Storm比较

在时效性上比storm弱，在吞吐量上比storm大 
streaming需要设置时间间隔，设置多长时间产生一个批次记录到streaming放在RDD里面 
比如设置5s，每隔5s就会产生一个RDD 
RDD需要是有序的

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fuax1z5n2wj30ne0ifgou.jpg)


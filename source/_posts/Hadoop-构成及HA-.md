---
title: Hadoop-HA-Federation机制
date: 2018年08月06日 22时15分52秒
tags: [Hadoop]
categories: 大数据
toc: true
---

[TOC]


（1）hadoop-HA集群运作机制介绍

所谓HA，即高可用（7*24小时不中断服务）实现高可用最关键的是消除单点故障，hadoop-ha严格来说应该分成各个组件的HA机制——HDFS的HA、YARN的HA

<!-- more -->

（2）HDFS的HA机制详解
通过双namenode消除单点故障
双namenode协调工作的要点：
​    A、元数据管理方式需要改变：
​    内存中各自保存一份元数据
​    Edits日志只能有一份，只有Active状态的namenode节点可以做写操作​，两个namenode都可以读取edits，共享的edits放在一个共享存储中管理（qjournal和NFS两个主流实现）
​    B、需要一个状态管理功能模块
​    实现了一个zkfailover，常驻在每一个namenode所在的节点
​    每一个zkfailover负责监控自己所在namenode节点，利用zk进行状态标识
​    当需要进行状态切换时，由zkfailover来负责切换
​    切换时需要防止brain split现象的发生

Hadoop-HA的主要思想是有两个NameNode，一个作为主NameNode，一个作为standby，两个NameNode使用同一个命名空间。通过zookeepr（JournalNode）来进行协调，实现NameNode的主备切换。

![](http://img.gangtieguo.cn/15361634728274.jpg)
真正的架构和流程如上图所示



# Hadoop中federation机制

共同运行多个active的namenode（多套主备的namenode集群），且公用用一套datanode

![](http://img.gangtieguo.cn/15361652893141.jpg)



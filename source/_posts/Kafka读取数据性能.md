---
title: Kafka读写数据
date: 2018年08月06日 22时15分52秒
tags: [Kafka,原理]
categories: 组件
toc: true
---

[TOC]

首先kafka依赖于操作系统的pageCache机制，尽可能的把空闲的内存作为一个磁盘，只有发生缺页的才会放在磁盘中

<!-- more -->

那么落在磁盘上的话，还有就是采用的是sendfile机制：主要思想就是在内核中进行拷贝，再写socket流，传统io是先将数据读到内核在写到用户区，在写到内核，再是socket流，这样就算磁盘的话，也是相当快的
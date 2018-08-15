---
title: Hadoop-HA
date: 2018年08月06日 22时15分52秒
tags: [Hadoop]
categories: 大数据
toc: true
---

[TOC]

Hadoop-HA的主要思想是有两个NameNode，一个作为主NameNode，一个作为standby，两个NameNode使用同一个命名空间。通过zookeepr（JournalNode）来进行协调，实现NameNode的主备切换。

<!-- more -->
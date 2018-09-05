---
title: Hadoop零碎知识点
date: 2018年08月06日 22时15分52秒
tags: [HDFS,原理,Hadoop]
categories: 大数据
toc: true
---

[TOC]

## 查看元数据信息
可以通过hdfs的一个工具来查看edits中的信息

```bash
bin/hdfs oev -i edits -o edits.xml
bin/hdfs oiv -i fsimage_0000000000000000087 -p XML -o fsimage.xml
```
<!-- more -->

查看目录树

```bash
hdfs会在配置文件中配置一个datanode的工作目录元数据 
查看目录结构 tree hddata/ 
```




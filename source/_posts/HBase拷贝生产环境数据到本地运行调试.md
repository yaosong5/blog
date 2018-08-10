---
title: HBase拷贝生产环境数据到本地运行调试
date: 2018年08月06日 22时15分52秒
tags: [HBase，操作]
categories: 大数据
toc: true
---

[TOC]

> 由于线上环境要经过跳板机跳转，并且打包测试，上传jar包步骤多，不然的话，要进行各种端口转发，且有权限控制，不易在本地idea编辑器上进行程序运行及调试
>
> 现在想法是，将线上测试环境的数据拷贝小部分到本地自己搭建的集群，进行程序的逻辑和初期调试
>
> 此贴就是记录一些操作

这都是要基于本地有HBASE及其依赖组件的。

主要思路是，拷贝线上查询的结果到文件hbaseout1.txt，将hbaseout1.txt文件sz导入本地

再在本地集群上将数据插入到hbase

<!-- more -->

# 1、创建和线上同名通结构的表

在线上执行 `describe 'beehive:a_up_rawdata'`

得到

![](https://ws4.sinaimg.cn/large/0069RVTdgy1fu4t1ryqfbj31kw074n4w.jpg)



在本地执行

```
hbase shell
create_namespace 'beehive'

create 'beehive:a_up_rawdata',{NAME => 'cf', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', COMPRESSION => 'NONE', TTL => 'FOREVER', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '1'}
```



# 2、拷贝线上hbase数据



## 查询结果导入文件

在线上机器任意目录执行

```bash
echo "get 'beehive:a_up_rawdata','530111199211287371',{COLUMN=>'cf:273468436_data'}"| hbase shell> hbaseout1.txt
```



解析: `get 'beehive:a_up_rawdata','530111199211287371',{COLUMN=>'cf:273468436_data'}`是执行的hbase的查询语句，将查询的结果存入到当前目录 **hbaseout1.txt**文件中

## 查询文件下载到本地

`sz hbaseout1.txt`

## 修改文件内容

可以查看hbaseout1.txt中可以看到会有表头

![](https://ws2.sinaimg.cn/large/0069RVTdgy1fu4s99k250j31g208qgs4.jpg)

需要将这部分表头数据删除，组成标准的导入文件

## 修改文件编码

![](https://ws1.sinaimg.cn/large/0069RVTdgy1fu4shn9bpnj31l203i0tv.jpg)

在通过 hbase shell 查看中文值时, 是 unicode 编码格式，使得直接查看中文值不太方便。如果要查看需要把 unicode 编码进行 decode

[参考]: https://blog.csdn.net/zychun1991/article/details/69938992	"hbase shell 中文 unicode 编码"

将查询结果导出来

```bash
print ('需要转码内容'.decode('utf-8'))

命令样例
python 2.7 
	print ('***\xE4\xBD\xA010009 '.decode('utf-8'))
python 3 
	print '***\xE4\xBD\xA010009 '.decode('utf-8')
```

可以有更友好的将内容设置为文件名，然后将转码后重新写入到一个文件，后续会更新



转码过后，文字显示正确

![](https://ws4.sinaimg.cn/large/0069RVTdgy1fu4sp3u3w6j31ak0f27fi.jpg)



## 重新组合文件

由于导入到hbase命令为 **`格式：hbase [类][分隔符] [行键，列族][表] [导入文件`]

由于我这次导入的文件里面有“,”，所有将分隔符设置为“|”

更改后的文件格式为

![](https://ws2.sinaimg.cn/large/0069RVTdgy1fu4ssvedr6j319a030wel.jpg)

将文件上传到hdfs

```bash
hadoop fs -put hadoop fs -put hbaseout1.txt /local/
```

## 将数据导入到本地hbase

    hbase org.apache.hadoop.hbase.mapreduce.ImportTsv  -Dimporttsv.separator="|"  -Dimporttsv.columns=HBASE_ROW_KEY,cf:273468436_data beehive:a_up_rawdata /local/hbaseout2.txt


# 3、校验查看

在hue上查看hbase内容，显示有数据

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fu50q6kl65j31kg0kymxw.jpg)

在hbase shell 查看

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fu515ozpw6j31kw08b7dl.jpg)


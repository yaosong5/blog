---
title: Shuffle中的机制
date: 2018年08月06日 22时15分52秒
tags: [Hadoop,原理]
categories: 大数据
toc: true
---

[TOC]

官方的shuffle流程

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuhbmle6ksj30ff07dglu.jpg)

# 介绍一下shuffle的原理

提到MapReduce，就不得不提一下shuffle。

MapReduce 框架的核心步骤主要分两部分：Map 和Reduce，一个是独立并发，一个是汇聚。当你向MapReduce 框架提交一个计算作业时，它会首先把计算作业拆分成若干个Map 任务，然后分配到不同的节点上去执行，每一个Map 任务处理输入数据中的一部分，当Map 任务完成后，它会生成一些中间文件，这些中间文件将会作为Reduce 任务的输入数据。Reduce 任务的主要目标就是把前面若干个Map 的输出汇总到一起并输出。

<!-- more -->

 我们知道每个reduce task输入的key都是按照key排序的。但是每个map的输出只是简单的key-value而非key-valuelist，所以shuffle的工作就是将map输出转化为reducer的输入的过程。Shuffle过程是指map产生输出结果开始，包括系统执行分区partition，排序sort，聚合Combiner（如有）以及传送map的输出到Reducer作为输入的过程。

## shuffle过程分析

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuhbuwapflj30zc0j4mxv.jpg)





## map端

1. 从map段开始分析，当Map开始产生输出的时候，并不是简单吧数据写到磁盘，因为频繁的操作会导致性能严重下降，首先将数据写入到一个环形缓冲区（每个maptask都会有一个环形缓冲区，默认100M，可以通过io.sort.mb属性来设置具体的大小）并做一些预排序，以提升效率，当缓冲区中的数据量达到一个特定的阀值(io.sort.mb * io.sort.spill.percent，其中io.sort.spill.percent 默认是0.80，即默认为 80MB），溢写线程启动。

2. 系统将会启动一个后台线程把缓冲区中的内容spill 到磁盘。即会锁定这80MB的内存，执行溢写过程。Map task的输出结果还可以往剩下的20MB内存中写，互不影响。在spill过程中，Map的输出将会继续写入到缓冲区，但如果缓冲区已经满了，Map就会被阻塞直到spill完成。spill线程在把缓冲区的数据写到磁盘前，会对他

   进行一个**二次排序**，**首先根据数据所属的partition排序（快速排序），然后每个partition中再按Key排序**。输出包括一个**索引文件和数据文件**。

3. 如果设定了Combiner，将在排序输出的基础上进行。Combiner就是一个Mini Reducer，它在执行Map任务的节点本身运行，先对Map的输出作一次简单的Reduce，有些数据可能像这样：`“a”/1， “a”/1， “a”/1，`会合并成 `“a”/3`，使得更少的数据会被写入磁盘和传送到Reducer。

4. Spill文件保存在由mapred.local.dir指定的目录中，Map任务结束后删除。每当内存中的数据达到spill阀值的时候，都会产生一个新的spill文件，所以在Map任务写完他的最后一个输出记录的时候，可能会有多个spill文件，在Map任务完成前，所有的spill文件将会被归并排序为一个索引文件和数据文件。这是一个多路归并过程，最大归并路数由io.sort.factor 控制(默认是10)。比如：“aaa”从某个map task读取过来时值是5，从另外一个map 读取时值是8，因为它们有相同的key，所以得merge成group。什么是group。对于“aaa”就是像这样的：{“aaa”, [5, 8, 2, …]}，数组中的值就是从不同溢写文件中读取出来的，然后再把这些值加起来。请注意，因为merge是将多个溢写文件合并到一个文件，所以可能也有相同的key存在。如果设定了Combiner，会使用combiner来合并相同key，这是map端的结果。



## map端与reduce端的交互

Reduce是怎么知道从哪些TaskTrackers中获取Map的输出呢？当Map任务完成之后，会通知他们的父TaskTracker，告知状态更新，然后TaskTracker再转告JobTracker，这些通知信息是通过心跳通信机制传输的，因此针对以一个特定的作业，jobtracker知道Map输出与tasktrackers的映射关系。Reducer中有一个线程会间歇的向JobTracker询问Map输出的地址，直到把所有的数据都取到。在Reducer取走了Map输出之后，TaskTracker不会立即删除这些数据，因为Reducer可能会失败，他们会在整个作业完成之后，JobTracker告知他们要删除的时候才去删除



map端的所有工作结束后，最终生成的这个文件也存放在TaskTracker够得着的某个本地目录内。每个reduce task不断地通过RPC从JobTracker那里获取map task是否完成的信息，如果reduce task得到通知，获知某台TaskTracker上的map task执行完成，Shuffle的后半段过程开始启动。
简单地说，reduce task在执行之前的工作就是不断地拉取当前job里每个map task的最终结果，然后对从不同地方拉取过来的数据不断地做merge，也最终形成一个文件作为reduce task的输入文件。





## reduce端过程

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fuhbdybylfj30z40hyq3f.jpg)

1. Copy过程，简单地拉取数据。Reduce进程启动一些数据copy线程(Fetcher)，通过HTTP方式请求map task所在的TaskTracker获取map task的输出文件。因为map task早已结束，这些文件就归TaskTracker管理在本地磁盘中。
2. Merge阶段。这里的merge如map端的merge动作，只是数组中存放的是不同map端copy来的数值。Copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比map端的更为灵活，它基于JVM的heap size设置，因为Shuffle阶段Reducer不运行，所以应该把绝大部分的内存都给Shuffle用。这里需要强调的是，merge有三种形式：1)内存到内存 （默认不启用） 2)内存到磁盘  3)磁盘到磁盘。当内存中的数据量到达一定阈值，就启动内存到磁盘的merge，这个过程中如果你设置有Combiner，也是会启用的，然后在磁盘中生成了众多的溢写文件。第二种merge方式一直在运行，直到没有map端的数据时才结束，然后启动第三种磁盘到磁盘的merge方式生成最终的那个文件。
3. Reducer的输入文件。不断地merge后，最后会生成一个文件。这个文件可能存在于磁盘上，也可能存在于内存中。就读取速度来说对于内存中，直接作为Reducer的输入，但默认情况下，这个文件是存放于磁盘中的。当Reducer的输入文件已定，整个Shuffle才最终结束。然后就是Reducer执行，一般是把结果放到HDFS上。





### Speculative Execution

是指当一个job的所有task都在running的时候，当某个task的进度比平均进度慢时才会启动一个和当前Task一模一样的任务，当其中一个task完成之后另外一个会被中止，所以Speculative Task不是重复Task而是对Task执行时候的一种优化策略





# 任务分片与hdfs文件大小及文件块之间的关系

在上面mapReduce中可以看到，任务的执行是基于hdfs文件的，任务分片和文件大小，文件块大小都有一定的联系。

任务切片：将任务划分成切片**，一个切片交给一个task实例**处理，只是一个逻辑的偏移量划分而已

1.在map task执行时，它的输入数据来源于HDFS的block，当然在MapReduce概念中，map task只读取split。Split与block的对应关系可能是多对一，默认是一对一。

采用的算法是：

**分片大小范围可以在 mapred-site.xml 中设置，mapred.min.split.size mapred.max.split.size，**minSplitSize 大小默认为 1B，**maxSplitSize 大小默认为 Long.MAX_VALUE = 9223372036854775807**

```java
miniSize = 1
maxSize = Long.MAXVALUE
splitSize = Math.max(miniSize,Math.min(maxSize,blockSize))
```

默认情况下，任务分片的大小为hdfs的blocksize 也就是块大小

**所以在我们没有设置分片的范围的时候，分片大小是由 block 块大小决定的，和它的大小一样。比如把一个 258MB 的文件上传到 HDFS 上，假设 block 块大小是 128MB，那么它就会被分成三个 block 块，与之对应产生三个 split**，所以最终会产生三个 map task。我又发现了另一个问题，第三个 block 块里存的文件大小只有 2MB，而它的 block 块大小是 128MB，那它实际占用 Linux file system 的多大空间？**

**答案是实际的文件大小，而非一个块的大小**



切片的流程

1. 遍历输入目录下的文件，得到文件集合 list

2. 遍历文件集合list，循环操作集合下的文件

   获取文件的blocksize 文件块，获取文件的长度，得到切片信息（split[文件路径,切片编号,偏移量范围]）,将各切片对象放入到一个splitList里面


3. 遍历完成后，将切片信息splitList序列化到一个split描述文件中

   ![](https://ws1.sinaimg.cn/large/006tNbRwgy1fufyt54ouvj30cq03c0so.jpg)





默认情况下，默认的TextInputFormat对任务切片是按文件规划切片，不管文件多小，都会是一个单独的切片，这样如果有大量小文件，就会产生大量的maptask，处理效率极其低下



对于以上原理，如果读取的是很多小文件，会产生大量的小切片，造成大量的maptask运行，对应的解决方法：

1. 将小文件合并之后再上传到hdfs
2. 如果小文件已经上传了，可以写MapReduce程序将小文件合并
3. 可以用另一种InputFormat：CombineInputFormat(它可以将多个文件划分到一个切片中)






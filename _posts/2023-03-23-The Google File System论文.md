---
layout: post
title: "The Google File System论文"
date: 2023-03-23
description: "The Google File System论文"
tag: 分布式系统
---

### 前言

回顾这篇2003年的论文，GFS可以说是“技术上辉煌而工程上保守”。说GFS技术上辉煌，是因为Google通过廉价的PC级别的硬件，搭建出了可以处理整个互联网网页数据的系统。而说GFS工程上保守，则是因为GFS没有“发明”什么特别的黑科技，而是在工程上做了大量的取舍（trade-off）。

### 目录

* [GFS的设计决策](#chapter1)
* [Wisckey设计目标](#chapter2)
* [并行Range查询](#chapter3)
* [Crash一致性](#chapter4)
* [GC机制](#chapter5)
* [改进点](#chapter6)

### <a name="chapter1"></a> 一、GFS的设计决策

##### <font color="#0000dd">三个原则</font><br /> #####
##### 1. 以工程上“简单”为设计原则  #####
GFS直接使用了Linux服务上的`(1)普通文件系统作为基础存储层`，并且选择了最简单的`(2)单Master`设计。单Master让GFS的架构变得非常简单，避免了需要管理复杂的一致性问题。不过它也带来了很多限制，比如一旦Master出现故障，整个集群就无法写入数据，所以GFS其实算不上一个`高可用`的系统。

但另外一方面，GFS还是采用了`Checkpoints`、`操作日志（Operation Logs）`、`影子 Master（Shadow Master）`等一系列的工程手段，来尽可能地保障整个系统的`可恢复（Recoverable）`，以及读层面的`可用性（Availability）`。

##### 2. 根据硬件特性来进行设计取舍  #####
2003年，大家都还在用机械硬盘，随机读写的性能很差，所以在GFS的设计中，重视的是顺序读写的性能，对随机写入的一致性甚至没有任何保障。

##### 3. 根据实际应用的特性，放宽了数据一致性（consistency）的选择  #####
论文里也提到，GFS是为了在廉价硬件上进行大规模数据处理而设计的。所以GFS的一致性相当宽松。GFS本身对于`随机写入的一致性没有任何保障`，而是把这个任务交给了客户端。对于`追加写入（Append）`，GFS 也只是作出了`至少一次（At LeastOnce）`这样宽松的保障。


### <a name="chapter2"></a>二、以“简单”为原则

![](https://s3.uuu.ovh/imgs/2023/03/25/bd84e804221f7bb4.png)

在这个设计原则下，可以看到GFS是一个非常简单的`单Master`架构，但是这个Master其实有三种不同的身份，分别是：

- 相对于存储数据的`Chunkserver`，Master是一个目录服务；
- 相对于为了灾难恢复的`Backup Master`，它是一个`同步复制的主从架构`下的主节点；
- 相对于为了保障读数据的可用性而设立的`Shadow Master`，它是一个`异步复制的主从架构`下的主节点。

并且，这三种身份是依靠不同的独立模块完成的，互相之间并不干扰。本节的内容主要介绍常见的`主从架构（Master-Slave）`下的Master的职责，以及数据复制的模式。

##### 1. Master 的第一个身份：一个目录服务  #####








### <a name="chapter3"></a>并行Range查询

Range-Query允许用户查询一段有序range的kv对，可以对其进行遍历。当发起一起Range-Query的时候，最终会被转换为vlog的多次随机读。相比于LSM-Tree的顺序读，value小的情况反而增加了延迟。因此在这种场景下，Wisckey会通过多个后台线程对后面的多个数据进行预读并缓存，充分利用SSD内部的并行性。预读缓存对于大范围的range-query效果比较明显，但是对于小范围的range-query可能作用不大。根据论文的实验数据，当value大于4KB时，读的性能才体验出来。

### <a name="chapter4"></a>Crash一致性

**思考第一个问题**：既然也会写入key到vlog里面，那么还需要LSM-Tree的WAL吗？

Wisckey的做法是去掉了LSM-Tree的WAL，直接使用vlog来代替，因为vlog里面存储了完整的key-value数据。`更细的写入流程是`：

1. 把key-value都全部append写入vlog。
2. 把key-value偏移位置写入LSM-Tree的memtable。
3. LSM-Tree会按照一定策略把memtable做Minor-Compaction写入sstable。

**思考另一个问题**：key-value分离存储后如何保证key-value写入的一致性？

因为Wisckey是先写入vlog，再写入LSM-Tree，所以会有以下几种失败情况：

* `vlog写入失败，LSM-Tree肯定写入失败`：可能会残留部分vlog数据，会有GC机制来回收。
* `vlog写入成功，LSM-Tree写入失败`：此时vlog的值便成了垃圾数据，会有GC机制来回收。
* `vlog写入成功，LSM-Tree写入成功后立刻崩溃`：因为已经移除了LSM-Tree的WAL，所以写入LSM-Tree也仅仅是写入了内存的memtable，此时程序或者机器发生崩溃LSM-Tree的数据依旧会丢失。Wisckey的做法是保存一个key-value对`<head, head-vLog-offset>`在LSM-Tree中。每次Open数据库的时候，读取head的值，然后依次读取head-vLog-offset之后的记录，并将key-value偏移位置写入到LSM-Tree，最后更新head的值。如果更新head的值发生崩溃也无所谓，只不过再次Open数据库的时候多重放一点记录罢了。

### <a name="chapter5"></a>GC机制

由于删除操作只会删除LSM-Tree里面的key-value偏移位置，不会删除vlog的value，同时在Crash一致性章节提出的各种Crash Case也都需要GC机制来做垃圾数据清理。

Wisckey除了会存储`<head, head-vLog-offset>`外，还会存储`<tail, tail-vLog-offset>`，用来标识最后做GC的位置。会从tail开始扫描vlog，获取key-value。为了不影响在线服务，会每次取出vlog最旧的一部分数据(几MB)，通过读取LSM-Tree来验证其有效性，如果过期或者被删除则丢弃；有效则append写入到最新的vlog，然后更新tail的值，并删除tail之前的数据，通过 `fallocate()`释放无效的文件磁盘空间。

### <a name="chapter6"></a>改进点

Wisckey key-value分离在大value下效果显著，但是对于小value却不甚友好，不如LSM-Tree。为此我们可以设置一个value阈值，当超过阈值时value存入vlog，小于阈值时存入LSM-Tree，提高小value场景的性能，BadgerDB正是这么做的。

### 参考资源

* [WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

* [转载自史明亚的博客](https://shimingyah.github.io/2019/08/BadgerDB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bwisckey%E8%AE%BA%E6%96%87/)

转载请注明：[sizheluo的博客](https://sizheluo.github.io) » [RocksDB源码分析之Wisckey论文](https://sizheluo.github.io/2023/03/RocksDB源码分析之Wisckey论文/)
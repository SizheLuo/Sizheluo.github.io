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
* [GFS 系统架构与设计](#chapter2)
* [GFS 读流程](#chapter3)
* [GFS 写流程](#chapter4)
* [GC机制](#chapter5)
* [改进点](#chapter6)

### <a name="chapter1"></a> 1. GFS 的设计决策

##### <font color="#0000dd">三个原则</font><br /> #####
##### 1. 以工程上“简单”为设计原则  #####
GFS直接使用了Linux服务上的`(1)普通文件系统作为基础存储层`，并且选择了最简单的`(2)单Master`设计。单Master让GFS的架构变得非常简单，避免了需要管理复杂的一致性问题。不过它也带来了很多限制，比如一旦Master出现故障，整个集群就无法写入数据，所以GFS其实算不上一个`高可用`的系统。

但另外一方面，GFS还是采用了`Checkpoints`、`操作日志（Operation Logs）`、`影子Master（Shadow Master）`等一系列的工程手段，来尽可能地保障整个系统的`可恢复（Recoverable）`，以及读层面的`可用性（Availability）`。

##### 2. 根据硬件特性来进行设计取舍  #####
2003年，大家都还在用机械硬盘，随机读写的性能很差，所以在GFS的设计中，重视的是顺序读写的性能，对随机写入的一致性甚至没有任何保障。

##### 3. 根据实际应用的特性，放宽了数据一致性（consistency）的选择  #####
论文里也提到，GFS是为了在廉价硬件上进行大规模数据处理而设计的。所以GFS的一致性相当宽松。GFS本身对于`随机写入的一致性没有任何保障`，而是把这个任务交给了客户端。对于`追加写入（Append）`，GFS 也只是作出了`至少一次（At LeastOnce）`这样宽松的保障。

### <a name="chapter2"></a>2. GFS 系统架构与设计

### <a name="chapter3"></a>2.1 GFS 集群的系统架构

一个 GFS cluster（集群）分为两个组件：

(1). `单个`master节点；

(2). `多个`chunkserver节点；

一个GFS集群同时可以被多个client（客户）节点访问。

一个 GFS 集群的架构可以用下图表示：

![](https://s3.uuu.ovh/imgs/2023/04/02/9d1f08752a4ee206.png)

可见GFS集群是一个典型Master + Worker结构。

	Master + Worker 结构说的是存在一个 Master 来管理任务、分配任务，而 Worker 是真正干活的节点。在这里干的活自然是数据的存储和读取。

### <a name="chapter3"></a>2.2 分块、三副本机制

![](https://s3.uuu.ovh/imgs/2023/04/03/651312aedb3a6d14.png)

为什么要分块？这是分布式系统在各个工业领域广泛应用的原因：`并行操作`。

GFS 以 64 MB 为固定的 Chunk Size 大小。大的 chunk 有如下的`优点`：

（1）减少 Client 与 Master 服务器交互的次数，因为对同一块进行多次读写仅仅需要向 Master 服务器发出一次初始请求`（客户端缓存机制）`。这可以有效地减少 Master 的工作负载；

（2）减少了 GFS Client 与 GFS chunkserver 进行交互的数据开销，这是因为数据的读取具有连续读取的倾向，即读到 offset 的字节数据后，下一次读取有较大的概率读紧挨着 offset 数据的后续数据，chunk 的大尺寸相当于提供了一层缓存，减少了网络 I/O 的开销；

（3）减少了存储在主服务器上的元数据的大小，有利于实现元数据全缓存。

`缺点`：小数据量（比如仅仅占据一个 chunk 的文件，文件至少占据一个 chunk）的文件很多时，当很多 GFS Client 同时将 record 存储到该文件时就会造成局部的 hot spots 热点。

### <a name="chapter3"></a>2.3 单master节点设计

Master其实有三种不同的身份，分别是：

- 相对于存储数据的`Chunkserver`，Master是一个目录服务；
- 相对于为了灾难恢复的`Backup Master`，它是一个`同步复制的主从架构`下的主节点；
- 相对于为了保障读数据的可用性而设立的`Shadow Master`，它是一个`异步复制的主从架构`下的主节点。

并且，这三种身份是依靠不同的独立模块完成的，互相之间并不干扰。本节的内容主要介绍常见的`主从架构（Master-Slave）`下的Master的职责，以及数据复制的模式。

**（1）Master 的第一个身份：一个目录服务**

master 里面会存放三种主要的元数据（metadata）：

- 文件和chunk 的命名空间信息，也就是类似前面`/data/geektime/bigdata/gfs01`这样的路径和文件名；
- 这些文件被拆分成了哪几个chunk，也就是这个全路径文件名到多个`chunk handle`的映射关系；
- 这些 chunk 实际被存储在了哪些 chunkserver 上，也就是 chunk handle 到 chunkserver 的映射关系。

![](https://s3.uuu.ovh/imgs/2023/04/02/13f44c34eff63650.png)

三者之间的KV组合关系为：

	Table 1：
		· key：
			file name
		· value：
			an array of chunk handler (nv)
	Table 2：
		· key：
			chunk handler
		· value：
			a list of chunkserver(v)
			chunk version number(nv)
			which chunkserver is primary node(which means others are nomal chunkserver in the list)(v)
			lease expiration time(v)

在 GFS 系统中，每一个 chunk 的唯一识别符是 chunk handler （有些地方也叫chunk index）。

Table1 的具体数据结构基于 HashMap 比较好，Table2 的具体实现通过 B+Tree 比较好。因为前者是文件名称作为 key，这是 HashMap 擅长的，查找的效率在 O(1)。而客户端可能一次性涉及读取大字节的数据，因此可能需要一次读取多个 chunk 的数据，涉及多个 chank handler， 这意味着最终要根据范围进行查找，范围查找 HashMap 并不擅长，B+Tree 则更擅长。

GFS 系统中，为了加快响应客户端关于 metadata 数据的请求，因此会将 metadata 存储于内存中，但是因为内存是易失性存储体，因此还需要持久化操作。具体来说：

客户端向 Master 节点请求的 metadata 数据直接存储于 Master 的内存中，避免每次请求都需要进行磁盘 I/O；

Master 节点使用日志 + checkpoint 的方式来确保数据的持久化；

上面表的介绍中括号中的 nv 含义是 Non-Volatile，也就是非易失性的含义，即要求将数据存储到磁盘上，而 v 的含义便是 Volatile，这些数据不需要持久化，当 Master 节点重启时，通过和每一个 chunkserver 进行通信来初始化不需要持久化的数据。

Master 节点内存中的 Table1 与 Table2 中的 nv 数据会定期存储到 Master 的磁盘上（包括 shadow Master 节点）；

**（2）Master 的第二个身份：Backup Master **

在单master的设计下，master 节点的所有数据，都是保存在内存里的。这样 master 的性能才能跟得上几百个客户端的并发访问。

但是数据放在内存里带来的问题，就是一旦 master 挂掉，数据就会都丢了。所以，master 会通过记录操作日志和定期生成对应的 Checkpoints 进行持久化，也就是写到硬盘上。

这是为了确保在 master 里的这些数据，不会因为一次机器故障就丢失掉。当 master 节点重启的时候，就会先读取最新的 Checkpoints，然后重放（replay）Checkpoints 之后的操作日志，把 master 节点的状态恢复到之前最新的状态。这是最常见的存储系统会用到的可恢复机制。



当然，光有这些还不够，如果只是 master 节点重新启动一下，从 Checkpoints 和日志中恢复到最新状态自然是很快的。可要是 master 节点的硬件彻底故障了呢？

所以 GFS 还为 master 准备好了几个“备胎”，也就是另外几台 Backup Master。所有针对 master 的数据操作，都需要同样写到另外准备的这几台服务器上。只有当数据在 master 上操作成功，对应的操作记录刷新到硬盘上，并且这几个 Backup Master 的数据也写入成功，并把操作记录刷新到硬盘上，整个操作才会被视为操作成功。

这种方式，叫做数据的`同步复制`，是分布式数据系统里的一种典型模式。假如你需要一个高可用的 MySQL 集群，一样也可以采用同步复制的方式，在主从服务器之间同步数据。

![](https://s3.uuu.ovh/imgs/2023/04/03/9cd0f59f74243b96.png)

而在同步复制这个机制之外，在集群外部还有监控 master 的服务在运行。如果只是 master 的进程挂掉了，那么这个监控程序会立刻重启 master 进程。而如果 master 所在的硬件或者硬盘出现损坏，那么这个监控程序就会在前面说的 Backup Master 里面找一个出来，启动对应的 master 进程，让它“备胎转正”，变成新的 master。而这个里面的数据，和原来的 master 其实一模一样。

不过，为了让集群中的其他 chunkserver 以及客户端不用感知这个变化，GFS 通过一个规范名称（Canonical Name）来指定 master，而不是通过 IP 地址或者 Mac 地址。这样，一旦要切换 master，这个监控程序只需要修改 DNS 的别名，就能达到目的。有了这个机制，GFS 的 master 就从之前的可恢复（Recoverable），进化成了能够快速恢复（Fast Recovery）。

不过，就算做到了快速恢复，我们还是不满足。毕竟，从监控程序发现 master 节点故障、启动备份节点上的 master 进程、读取 Checkpoints 和操作日志，仍然是一个几秒级别乃至分钟级别的过程。在这个时间段里，我们可能仍然有几百个客户端程序“嗷嗷待哺”，希望能够在 GFS 上读写数据。虽然作为单个 master 的设计，这个时候的确是没有办法去写入数据的。

**（3）Master 的第三个身份：Shadow Master **

为了让我们这个时候还能够从 GFS 上读取数据。加入一系列只读的`影子 Master`，这些影子 Master 和前面的备胎不同，master 写入数据并不需要等到影子 Master 也写入完成才返回成功。而是影子 Master 不断同步 master 输入的写入，尽可能保持追上 master 的最新状态。

这种方式，叫做数据的`异步复制`，是分布式系统里另一种典型模式。异步复制下，影子 Master 并不是和 master 的数据完全同步的，而是可能会有一些小小的延时。

影子 Master 会不断同步 master 里的数据，不过当 master 出现问题的时候，客户端们就可以从这些影子 Master 里找到自己想要的信息。当然，因为小小的延时，客户端有很小的概率，会读到一些过时的 master 里面的信息，比如命名空间、文件名等这些元数据。



但你也要知道，这种情况其实只会发生在以下三个条件都满足的情况下：

第一个，是 master 挂掉了；

第二个，是挂掉的 master 或者 Backup Master 上的 Checkpoints 和操作日志，还没有被影子 Master 同步完；

第三个，则是我们要读取的内容，恰恰是在没有同步完的那部分操作上；

![](https://s3.uuu.ovh/imgs/2023/04/03/95e4b5c963b80976.png)

相比于这个小小的可能性，影子 Master 让整个 GFS 在 master 快速恢复的过程中，虽然不能写数据，但仍然是完全可读的。至少在集群的读取操作上，GFS 可以算得上是`高可用（High Availability）`的了。

### <a name="chapter2"></a>3. GFS 读流程

![](https://s3.uuu.ovh/imgs/2023/04/02/9d1f08752a4ee206.png)


（1）客户端先去问 master，我们想要读取的数据在哪里。这里，客户端会发出两部分信息，一个是文件名，另一个则是要读取哪一段数据，也就是读取文件的 offset 及 length。因为所有文件都被切成 64MB 大小的一个 chunk 了，所以根据 offset 和 length，我们可以很容易地算出客户端要读取的数据在哪几个 chunk 里面。于是，客户端就会告诉 master，我要哪个文件的第几个 chunk。

（2）master 拿到了这个请求之后，就会把这个 chunk 对应的所有副本所在的 

（3）chunkserver，告诉客户端。

（4）等客户端拿到 chunk 所在的 chunkserver 信息后，客户端就可以直接去找其中任意的一个 chunkserver 读取自己所要的数据。





### <a name="chapter2"></a>4. GFS 写流程


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

* [Spongecaptain's Blog](https://spongecaptain.cool/post/paper/googlefilesystem/#34-read-%E8%AF%BB%E6%93%8D%E4%BD%9C)

转载请注明：[sizheluo的博客](https://sizheluo.github.io) » [RocksDB源码分析之Wisckey论文](https://sizheluo.github.io/2023/03/RocksDB源码分析之Wisckey论文/)
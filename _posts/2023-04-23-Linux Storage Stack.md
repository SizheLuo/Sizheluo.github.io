---
layout: post
title: "Linux Storage Stack"
date: 2023-04-23
description: "linux io过程自顶向下分析"
tag: 操作系统
---

### 前言

IO是操作系统内核最重要的组成部分之一，它的概念很广，本文主要针对的是普通文件与设备文件的IO原理，采用自顶向下的方式，去探究从用户态的fread,fwrite函数到底层的数据是如何被读出和写入磁盘的的全过程。希望通过本文，能梳理出一条清晰的linux io过程的脉络。

### 目录

* [文件与存储](#chapter1)
* [linux io体系结构](#chapter2)
* [非阻塞IO模型](#chapter3)
* [IO复用模型](#chapter4)
* [信号驱动IO模型](#chapter5)
* [异步IO模型](#chapter6)

### <a name="chapter1"></a>文件与存储

为了解决数据存储与持久化问题，有很多不同类型的文件系统（ext2,ext3,ext4,xfs,…)，它们大多数是工作在内核态的，也有少数的用户态文件系统（fuse）。linux为了便于管理这些不同的文件系统，提出了一个虚拟文件系统（VFS）的概念，对这些不同的文件系统提供了一套统一的对外接口。本文将会从虚拟文件系统说起，自顶向下地去阐述io的每一层的概念。

![](https://wjqwsp.github.io/img/mmap.png)

### <a name="chapter2"></a>linux io体系结构

![](https://raw.githubusercontent.com/zjs1224522500/PicGoImages/master//img/blog/20210422102901.png)

上图中使用了不同的颜色来区分组件（从下往上）：  

- 天蓝色：硬件存储设备  
- 橙色：传输协议  
- 蓝色：Linux系统中的设备文件  
- 黄色：I/O 调度策略  
- 绿色：Linux 中的文件系统  
- 蓝绿色：Linux 存储操作基本数据结构 bio  

上图也有简化版：

![](https://raw.githubusercontent.com/zjs1224522500/PicGoImages/master//img/blog/20210421154439.png)

![](https://wjqwsp.github.io/img/io-structure.png)

### <a name="chapter3"></a>2、非阻塞IO模型

通过进程反复调用IO函数，在数据拷贝过程中，进程是阻塞的。模型图如下所示:

![](https://s3.uuu.ovh/imgs/2023/04/08/6e6b3e864487507a.png)

### <a name="chapter4"></a>3、IO复用模型

主要是select和epoll。一个线程可以对多个IO端口进行监听，当socket有读写事件时分发到具体的线程进行处理。模型如下所示：

![](https://s3.uuu.ovh/imgs/2023/04/08/6ebb27489b5668cb.png)

### <a name="chapter5"></a>4、信号驱动IO模型

信号驱动式I/O：首先我们允许Socket进行信号驱动IO,并安装一个信号处理函数，进程继续运行并不阻塞。当数据准备好时，进程会收到一个SIGIO信号，可以在信号处理函数中调用I/O操作函数处理数据。过程如下图所示：

![](https://s3.uuu.ovh/imgs/2023/04/08/04935453025bdbb8.png)

### <a name="chapter6"></a>5、异步IO模型

相对于同步IO，异步IO不是顺序执行。用户进程进行aio_read系统调用之后，无论内核数据是否准备好，都会直接返回给用户进程，然后用户态进程可以去做别的事情。等到socket数据准备好了，内核直接复制数据给进程，然后从内核向进程发送通知。IO两个阶段，进程都是非阻塞的。异步过程如下图所示：

![](https://s3.uuu.ovh/imgs/2023/04/08/2ca15f43fc8b9e37.png)

### 参考资源

* [南国倾城](https://wjqwsp.github.io/2018/12/18/linux-io%E8%BF%87%E7%A8%8B%E8%87%AA%E9%A1%B6%E5%90%91%E4%B8%8B%E5%88%86%E6%9E%90/)

* [Elvis Zhang](https://blog.shunzi.tech/post/Linux_Storage_Stack/)

转载请注明：[sizheluo的博客](https://sizheluo.github.io) » [Linux下的五种IO模型](https://sizheluo.github.io/2023/04/Linux-IO%E6%A8%A1%E5%9E%8B//)
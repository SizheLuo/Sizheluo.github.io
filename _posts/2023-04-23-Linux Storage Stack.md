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
* [从Hello world说起](#chapter3)
* [虚拟文件系统（VFS）](#chapter4)
* [信号驱动IO模型](#chapter5)
* [异步IO模型](#chapter6)
* [信号驱动IO模型](#chapter7)
* [异步IO模型](#chapter8)
* [信号驱动IO模型](#chapter9)
* [异步IO模型](#chapter10)

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

本文将按照上图的架构自顶向下依次分析每一层的要点。

### <a name="chapter3"></a>从Hello world说起

	#include "apue.h"
	      
	      #define BUFFSIZE        4096
	       
	      int
	      main(void)
	      {
	              int             n;
	              char    buf[BUFFSIZE];
	              int fd1;
	              int fd2;
	             
	              fd1 = open("helloworld.in", O_RONLY);
	              fd2 = open("helloworld.out", O_WRONLY);
	              while ((n = read(fd1, buf, BUFFSIZE)) > 0)
	                     if (write(fd2, buf, n) != n)
	                             err_sys("write error");
	     
	              if (n < 0)
	                      err_sys("read error");
	      
	              exit(0);
	     }

　　看一个简单的hello world程序，它的功能非常简单，就是在栈空间里分配4096个字节作为buffer，从helloworld.in文件里读4KB到该buffer里，然后将该buffer的数据写到helloworld.out文件中。这个hello world进程是工作于用户态的，但由于操作系统的隔离性，用户态程序是无法直接操作硬件的，所以要通过`read`, `write`系统调用进入内核态，执行内核的相应代码。

　　现在我们看看`read`, `write`系统调用做了什么。

	SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
	{
		struct fd f = fdget_pos(fd);
		ssize_t ret = -EBADF;
		if (f.file) {
			loff_t pos = file_pos_read(f.file);
			ret = vfs_read(f.file, buf, count, &pos);
			file_pos_write(f.file, pos);
			fdput_pos(f);
		}
		return ret;
	}

	SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
			size_t, count)
	{
		struct fd f = fdget_pos(fd);
		ssize_t ret = -EBADF;
		if (f.file) {
			loff_t pos = file_pos_read(f.file);
			ret = vfs_write(f.file, buf, count, &pos);
			file_pos_write(f.file, pos);
			fdput_pos(f);
		}
		return ret;
	}

　`read`, `write`系统调用实际上就是对`vfs_read`和`vfs_write`的一个封装，非常直接地进入了`虚拟文件系统（VFS）`这一层。

### <a name="chapter4"></a>虚拟文件系统（VFS）

　　在深入`vfs_read`和`vfs_write`函数之前，必须先介绍一下`vfs`的基本知识。

　　`vfs`为不同的文件系统提供了统一的对上层的接口，使得上层应用在调用的时候无需知道底层的具体文件系统，只有在真正执行读写操作的时候才调用相应文件系统的读写函数。这个思想与面向对象编程的多态思想是非常相似的。实际上，`vfs`就是定义了4种类型的基本对象，不同的文件系统要做的工作就是去实现具体的这4种对象。下面介绍一下这4种对象。这4种对象分别是`superblock`, `inode`, `dentry`和`file`。`file`对象就是我们用户进程打开了一个文件所产生的，每个`file`对象对应一个唯一的`dentry`对象。`dentry`代表目录项，是跟文件路径相关的，一个文件路径对应唯一的一个`dentry`。`dentry`与`inode`则是多对一的关系，`inode`存放单个文件的元信息，由于文件可能存在链接，所以多个路径的文件可能指向同一个`inode`。`superblock`对象则是整个文件系统的元信息，它还负责存储空间的分配与管理。它们的关系可以用下图表示：

![](https://wjqwsp.github.io/img/vfs-model.png)

#####superblock对象#####

　　superblock对象定义了整个文件系统的元信息，它实际上是一个结构体。

[https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_super.c#L1789](https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_super.c#L1789)

```
static const struct super_operations xfs_super_operations = {
	......
};

static struct file_system_type xfs_fs_type = {
	.name			= "xfs",
	......
};
```

　　这里字段非常多，我们没必要一一解释，有个大概的感觉就行。有几个字段比较重要的这里提一下：

1. `s_list`该字段是双向循环链表相邻元素的指针，所有的superblock对象都以链表的形式链在一起。  
2. `s_fs_info`字段指向属于具体文件系统的超级块信息。很多具体的文件系统，例如ext2，在磁盘上有对应的superblock的数据块，为了访问效率，`s_fs_info`就是该数据块在内存中的缓存。这个结构里最重要的是用bitmap形式存放了所有磁盘块的分配情况，任何分配和释放磁盘块的操作都要修改这个字段。  
3. `s_dirt`字段表示超级块是否是脏的，上面提到修改了s_fs_info字段后超级块就变成脏了，脏的超级块需要定期写回磁盘，所以当计算机掉电时候是有可能造成文件系统损坏的。
4. `s_dirty`字段引用了脏inode链表的首元素和尾元素。
5. `s_inodes`字段引用了该超级块的所有inode构成的链表的首元素和尾元素。
6. `s_op`字段封装了一些函数指针，这些函数指针就指向真实文件系统实现的函数，superblock对象定义的函数主要是读、写、分配inode的操作。多态主要是通过函数指针指向不同的函数实现的。

#####inode对象#####

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
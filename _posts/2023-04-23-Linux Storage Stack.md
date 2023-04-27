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
* [高速缓存](#chapter5)
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

#### superblock对象

　　superblock对象定义了整个文件系统的元信息，它实际上是一个结构体。

```
struct super_block {
	struct list_head	s_list;		/* Keep this first */
	dev_t			s_dev;		/* search index; _not_ kdev_t */
	unsigned char		s_blocksize_bits;
	unsigned long		s_blocksize;
	loff_t			s_maxbytes;	/* Max file size */
	struct file_system_type	*s_type;
	const struct super_operations	*s_op;
	const struct dquot_operations	*dq_op;
	const struct quotactl_ops	*s_qcop;
	const struct export_operations *s_export_op;
	unsigned long		s_flags;
	unsigned long		s_magic;
	struct dentry		*s_root;
	struct rw_semaphore	s_umount;
	int			s_count;
	atomic_t		s_active;
#ifdef CONFIG_SECURITY
	void                    *s_security;
#endif
	const struct xattr_handler **s_xattr;
	struct list_head	s_inodes;	/* all inodes */
	struct hlist_bl_head	s_anon;		/* anonymous dentries for (nfs) exporting */
#ifdef __GENKSYMS__
#ifdef CONFIG_SMP
	struct list_head __percpu *s_files;
#else
	struct list_head	s_files;
#endif
#else
#ifdef CONFIG_SMP
	struct list_head __percpu *s_files_deprecated;
#else
	struct list_head	s_files_deprecated;
#endif
#endif
	struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
	/* s_dentry_lru, s_nr_dentry_unused protected by dcache.c lru locks */
	struct list_head	s_dentry_lru;	/* unused dentry lru */
	int			s_nr_dentry_unused;	/* # of dentry on lru */
	/* s_inode_lru_lock protects s_inode_lru and s_nr_inodes_unused */
	spinlock_t		s_inode_lru_lock ____cacheline_aligned_in_smp;
	struct list_head	s_inode_lru;		/* unused inode lru */
	int			s_nr_inodes_unused;	/* # of inodes on lru */
	struct block_device	*s_bdev;
	struct backing_dev_info *s_bdi;
	struct mtd_info		*s_mtd;
	struct hlist_node	s_instances;
	struct quota_info	s_dquot;	/* Diskquota specific options */
	struct sb_writers	s_writers;
	char s_id[32];				/* Informational name */
	u8 s_uuid[16];				/* UUID */
	void 			*s_fs_info;	/* Filesystem private info */
	unsigned int		s_max_links;
	fmode_t			s_mode;
	/* Granularity of c/m/atime in ns.
	   Cannot be worse than a second */
	u32		   s_time_gran;
	/*
	 * The next field is for VFS *only*. No filesystems have any business
	 * even looking at it. You had been warned.
	 */
	struct mutex s_vfs_rename_mutex;	/* Kludge */
	/*
	 * Filesystem subtype.  If non-empty the filesystem type field
	 * in /proc/mounts will be "type.subtype"
	 */
	char *s_subtype;
	/*
	 * Saved mount options for lazy filesystems using
	 * generic_show_options()
	 */
	char __rcu *s_options;
	const struct dentry_operations *s_d_op; /* default d_op for dentries */
	/*
	 * Saved pool identifier for cleancache (-1 means none)
	 */
	int cleancache_poolid;
	struct shrinker s_shrink;	/* per-sb shrinker handle */
	/* Number of inodes with nlink == 0 but still referenced */
	atomic_long_t s_remove_count;
	/* Being remounted read-only */
	int s_readonly_remount;
	/* AIO completions deferred from interrupt context */
	RH_KABI_EXTEND(struct workqueue_struct *s_dio_done_wq)
};
```

　　这里字段非常多，我们没必要一一解释，有个大概的感觉就行。有几个字段比较重要的这里提一下：

1. `s_list`该字段是双向循环链表相邻元素的指针，所有的superblock对象都以链表的形式链在一起。  
2. `s_fs_info`字段指向属于具体文件系统的超级块信息。很多具体的文件系统，例如ext2，在磁盘上有对应的superblock的数据块，为了访问效率，`s_fs_info`就是该数据块在内存中的缓存。这个结构里最重要的是用bitmap形式存放了所有磁盘块的分配情况，任何分配和释放磁盘块的操作都要修改这个字段。  
3. `s_dirt`字段表示超级块是否是脏的，上面提到修改了s_fs_info字段后超级块就变成脏了，脏的超级块需要定期写回磁盘，所以当计算机掉电时候是有可能造成文件系统损坏的。
4. `s_dirty`字段引用了脏inode链表的首元素和尾元素。
5. `s_inodes`字段引用了该超级块的所有inode构成的链表的首元素和尾元素。
6. `s_op`字段封装了一些函数指针，这些函数指针就指向真实文件系统实现的函数，superblock对象定义的函数主要是读、写、分配inode的操作。多态主要是通过函数指针指向不同的函数实现的。

#### inode对象

　　inode对象定义了单个文件的元信息，例如最后修改时间、最后访问时间等等，同时还定义了一连串函数指针，这些函数指针指向具体文件系统的对文件操作的函数，可以说文件系统最核心的功能全部由inode的函数指针提供接口，包括常见的open,read,write,sync,close等等文件操作，都在inode的i_op字段里定义了统一的接口函数。

	struct inode {
		umode_t			i_mode;
		unsigned short		i_opflags;
		kuid_t			i_uid;
		kgid_t			i_gid;
		unsigned int		i_flags;
	#ifdef CONFIG_FS_POSIX_ACL
		struct posix_acl	*i_acl;
		struct posix_acl	*i_default_acl;
	#endif
		const struct inode_operations	*i_op;
		struct super_block	*i_sb;
		struct address_space	*i_mapping;
	#ifdef CONFIG_SECURITY
		void			*i_security;
	#endif
		/* Stat data, not accessed from path walking */
		unsigned long		i_ino;
		/*
		 * Filesystems may only read i_nlink directly.  They shall use the
		 * following functions for modification:
		 *
		 *    (set|clear|inc|drop)_nlink
		 *    inode_(inc|dec)_link_count
		 */
		union {
			const unsigned int i_nlink;
			unsigned int __i_nlink;
		};
		dev_t			i_rdev;
		loff_t			i_size;
		struct timespec		i_atime;
		struct timespec		i_mtime;
		struct timespec		i_ctime;
		spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
		unsigned short          i_bytes;
		unsigned int		i_blkbits;
		blkcnt_t		i_blocks;
	#ifdef __NEED_I_SIZE_ORDERED
		seqcount_t		i_size_seqcount;
	#endif
		/* Misc */
		unsigned long		i_state;
		struct mutex		i_mutex;
		unsigned long		dirtied_when;	/* jiffies of first dirtying */
		struct hlist_node	i_hash;
		struct list_head	i_wb_list;	/* backing dev IO list */
		struct list_head	i_lru;		/* inode LRU list */
		struct list_head	i_sb_list;
		union {
			struct hlist_head	i_dentry;
			struct rcu_head		i_rcu;
		};
		u64			i_version;
		atomic_t		i_count;
		atomic_t		i_dio_count;
		atomic_t		i_writecount;
		const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
		struct file_lock	*i_flock;
		struct address_space	i_data;
	#ifdef CONFIG_QUOTA
		struct dquot		*i_dquot[MAXQUOTAS];
	#endif
		struct list_head	i_devices;
		union {
			struct pipe_inode_info	*i_pipe;
			struct block_device	*i_bdev;
			struct cdev		*i_cdev;
		};
		__u32			i_generation;
	#ifdef CONFIG_FSNOTIFY
		__u32			i_fsnotify_mask; /* all events this inode cares about */
		struct hlist_head	i_fsnotify_marks;
	#endif
	#ifdef CONFIG_IMA
		atomic_t		i_readcount; /* struct files open RO */
	#endif
		void			*i_private; /* fs or device private pointer */
	};

　除了i_op外，介绍几个重要的字段：

1. `i_state`表示inode的状态，主要是表示inode是否是脏的。一般的文件系统在磁盘上都有相应的inode数据块，内核的inode结构便是这个磁盘数据块的内存缓存，所以与superblock一样，也是需要定期写回磁盘的，否则会导致数据丢失。  
2. `i_list`把操作系统里的某些inode用双向循环链表连接起来，该字段指向相应链表的前一个元素和后一个元素。内核中有好几个关于inode的链表，所有inode必定出现在其中的某个链表内。第一个链表是有效未使用链表，链表里的inode都是非脏的，并且没有被引用，仅仅是作为高速缓存存在。第二个链表是正在使用链表，inode不为脏，但i_count字段为整数，表示被某些进程引用了。第三个链表是脏链表，由superblock的s_dirty字段引用。
3. `i_sb_list`存放了超级块对象的s_inodes字段引用的链表的前一个元素和后一个元素。
4. 所有的inode都存放在一个inode_hashtable的哈希表中，key是inode编号和超级块对象的地址计算出来的，作为高速缓存。因为哈希表可能会存在冲突，`i_hash`字段也是维护了链表指针，就指向同一个哈希地址的前一个inode和后一个inode。

#### dentry对象

　　dentry代表目录项，因为每一个文件必定存在于某个目录内，我们通过路径去查找一个文件，最终必定最终找到某个目录项。在linux里，目录与普通文件一样，往往都是存放在磁盘的数据块中，在查找目录的时候就读出该目录所在的数据块，然后去寻找其中的某个目录项。如果不存在硬链接，其实dentry是没有必要的，仅仅通过inode就能确定文件。但多个路径有可能指向同一个文件，所以vfs还抽象出了一个dentry的对象，一个或多个dentry对应一个inode。

	struct dentry {
		/* RCU lookup touched fields */
		unsigned int d_flags;		/* protected by d_lock */
		seqcount_t d_seq;		/* per dentry seqlock */
		struct hlist_bl_node d_hash;	/* lookup hash list */
		struct dentry *d_parent;	/* parent directory */
		struct qstr d_name;
		struct inode *d_inode;		/* Where the name belongs to - NULL is
						 * negative */
		unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */
		/* Ref lookup also touches following */
		struct lockref d_lockref;	/* per-dentry lock and refcount */
		const struct dentry_operations *d_op;
		struct super_block *d_sb;	/* The root of the dentry tree */
		unsigned long d_time;		/* used by d_revalidate */
		void *d_fsdata;			/* fs-specific data */
		struct list_head d_lru;		/* LRU list */
		/*
		 * d_child and d_rcu can share memory
		 */
		union {
			struct list_head d_child;	/* child of parent list */
		 	struct rcu_head d_rcu;
		} d_u;
		struct list_head d_subdirs;	/* our children */
		struct hlist_node d_alias;	/* inode alias list */
	};

　　介绍几个重要的字段：

1. `d_inode`指向该dentry对应的inode，找到了dentry就可以通过它找到inode。
2. 同一个inode的所有dentry都在一个链表内，`d_alias`指向该链表的前一个和后一个元素。
3. `d_op`定义了一些关于目录项的函数指针，指向具体文件系统的函数。

　　dentry必须放在高速缓存中加速。vfs把所有的dentry都放到dentry_hashtable这个哈希表中，key就是由目录项对象和文件名哈希产生。目录项高速缓存采用LRU双向链表来进行缓存添加与释放。

#### file对象

　　作为应用程序的开发者或使用者，我们平时能接触到的vfs对象就只有`file`对象。我们平常说的打开文件，实际上就是让内核去创建一个`file`对象，并返回给我们一个文件描述符。出于隔离性的考虑，内核不可能把`file`对象的地址传给我们，我们只能通过文件描述符去间接地访问`file`对象。

	struct file {
		/*
		 * fu_list becomes invalid after file_free is called and queued via
		 * fu_rcuhead for RCU freeing
		 */
		union {
			struct list_head	fu_list;
			struct rcu_head 	fu_rcuhead;
		} f_u;
		struct path		f_path;
	#define f_dentry	f_path.dentry
		struct inode		*f_inode;	/* cached value */
		const struct file_operations	*f_op;
		/*
		 * Protects f_ep_links, f_flags.
		 * Must not be taken from IRQ context.
		 */
		spinlock_t		f_lock;
	#ifdef __GENKSYMS__
	#ifdef CONFIG_SMP
		int			f_sb_list_cpu;
	#endif
	#else
	#ifdef CONFIG_SMP
		int			f_sb_list_cpu_deprecated;
	#endif
	#endif
		atomic_long_t		f_count;
		unsigned int 		f_flags;
		fmode_t			f_mode;
		loff_t			f_pos;
		struct fown_struct	f_owner;
		const struct cred	*f_cred;
		struct file_ra_state	f_ra;
		u64			f_version;
	#ifdef CONFIG_SECURITY
		void			*f_security;
	#endif
		/* needed for tty driver, and maybe others */
		void			*private_data;
	#ifdef CONFIG_EPOLL
		/* Used by fs/eventpoll.c to link all the hooks to this file */
		struct list_head	f_ep_links;
		struct list_head	f_tfile_llink;
	#endif /* #ifdef CONFIG_EPOLL */
		struct address_space	*f_mapping;
	#ifdef CONFIG_DEBUG_WRITECOUNT
		unsigned long f_mnt_write_state;
	#endif
	#ifndef __GENKSYMS__
		struct mutex		f_pos_lock;
	#endif
	};

1. `f_inode`指向对应的inode对象。
2. `f_dentry`指向对应的dentry对象。
3. `f_pos`表示当前文件的偏移，可见文件偏移是每个file对象都有自己的独立的文件偏移量。
4. `f_op`表示当前文件相关的所有函数指针，实际上在文件open的时候f_op会全部赋值为`i_op`相应的函数指针。

　　由于file对象是由内核进程直接管理的，我们有必要了解一下进程如何管理打开的文件。首先，每个进程有一个`fs_struc`的字段：

	struct fs_struct {
		int users;
		spinlock_t lock;
		seqcount_t seq;
		int umask;
		int in_exec;
		struct path root, pwd;
	};
	struct path {
		struct vfsmount *mnt;
		struct dentry *dentry;
	};

　　我们看到，每个进程都维护一个根目录和当前工作目录的信息，每一个目录由`vfsmount`和`dentry`组合唯一确定，`dentry`代表目录项前面已经说到，`vfsmount`则代表相应目录项所在文件系统的挂载信息，会在后面展开介绍一下。

　　然后，每个进程都有当前打开的文件表，存放在进程的`files_struct`结构中。

	struct files_struct {
	  /*
	   * read mostly part
	   */
		atomic_t count;
		struct fdtable __rcu *fdt;
		struct fdtable fdtab;
	  /*
	   * written part on a separate cache line in SMP
	   */
		spinlock_t file_lock ____cacheline_aligned_in_smp;
		int next_fd;
		unsigned long close_on_exec_init[1];
		unsigned long open_fds_init[1];
		struct file __rcu * fd_array[NR_OPEN_DEFAULT];
	};

　　我们只要关注一下`fd_array`这个数组就行，这个数组就存储了所有打开的file对象，我们应用程序拿到的文件描述符实际上就只是这个数组的索引。

#### vfs管理文件系统的注册与挂载

　　前面介绍了vfs的4种对象，所有能在linux上使用的文件系统都必须实现这4种对象接口，实际上就是实现`s_op`, `i_op`, `d_op`这3组函数（`f_op`简单地复制`i_op`），这样vfs才能正常地使用文件系统进行工作。假设我们已经按照这个接口开发了一套文件系统，这时候又面临了一个问题，vfs怎么去识别我们的文件系统呢？我们的文件系统是如何注册和挂载到vfs上的呢？

##### 文件系统注册

　　文件系统要么是固化在内核代码中的，要么是通过内核模块动态加载的，在内核代码中的随着操作系统启动会自动注册，而通过内核模块动态加载的也可以用操作系统的启动参数配置成自动注册，或者我们可以人为地执行类似这样的命令`insmod fuse.ko`去动态注册，这里的fuse.ko就是fuse文件系统编译链接出来的二进制文件。

	struct file_system_type {
		const char *name;
		int fs_flags;
	#define FS_REQUIRES_DEV		1 
	#define FS_BINARY_MOUNTDATA	2
	#define FS_HAS_SUBTYPE		4
	#define FS_USERNS_MOUNT		8	/* Can be mounted by userns root */
	#define FS_USERNS_DEV_MOUNT	16 /* A userns mount does not imply MNT_NODEV */
	#define FS_HAS_RM_XQUOTA	256	/* KABI: fs has the rm_xquota quota op */
	#define FS_HAS_INVALIDATE_RANGE	512	/* FS has new ->invalidatepage with length arg */
	#define FS_HAS_DIO_IODONE2	1024	/* KABI: fs supports new iodone */
	#define FS_HAS_NEXTDQBLK	2048	/* KABI: fs has the ->get_nextdqblk op */
	#define FS_HAS_DOPS_WRAPPER	4096	/* kabi: fs is using dentry_operations_wrapper. sb->s_d_op points to
	dentry_operations_wrapper */
	#define FS_RENAME_DOES_D_MOVE	32768	/* FS will handle d_move() during rename() internally. */
		struct dentry *(*mount) (struct file_system_type *, int,
			       const char *, void *);
		void (*kill_sb) (struct super_block *);
		struct module *owner;
		struct file_system_type * next;
		struct hlist_head fs_supers;
		struct lock_class_key s_lock_key;
		struct lock_class_key s_umount_key;
		struct lock_class_key s_vfs_rename_key;
		struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];
		struct lock_class_key i_lock_key;
		struct lock_class_key i_mutex_key;
		struct lock_class_key i_mutex_dir_key;
	};

　　在注册文件系统的时候，我们需要提交一个`file_system_type`,这个对象主要有一个`get_sb`（linux kernel 2.6）或者是`mount`（linux kernel 3.1）对象，这个是一个函数指针，主要是分配superblock的，每当该类型的文件系统挂载时，就会调用该函数分配superblock对象。`fs_supers`引用了所有属于该文件系统类型的superblock对象。内核把所有注册的文件系统类型维护成一个链表，`file_system_type`的`next`字段指向链表的下一个元素。

##### 文件系统挂载

　　正常情况下，操作系统启动后，常用的文件系统类型都是自动注册的，不需要用户干预。但一个块设备要以某文件系统的形式被操作系统识别的话，需要挂载到某个目录下，例如执行如下的挂载命令：

	mount -t xfs /dev/sdb /var/cold-storage/

　　当执行这条命令以后，内核会首先分配一个vfsmount的对象，该对象唯一标识一个挂载的文件系统。

	struct vfsmount {
		struct dentry *mnt_root;	/* root of the mounted tree */
		struct super_block *mnt_sb;	/* pointer to superblock */
		int mnt_flags;
	};

　　`vfsmount`主要存放了该文件系统的superblock对象以及该文件系统根目录（上例的/var/cold-storage/）的dentry对象，一开始superblock对象是空的。

　　有可能这个文件系统会被挂载了多次，之前已经被挂载到其他目录上了，就意味着其superblock对象已经被分配，因此内核会先搜索`file_system_type`的`fs_supers`链表，如果找到，就直接用该superblock对象赋值给新的`vfsmount`对象的`mnt_sb`字段。

　　如果这个文件系统是第一次被挂载，则调用注册的`file_system_type`的`get_sb`或者`mount`函数，分配新的superblock对象。

　　当superblock对象确定了以后，该文件系统就被挂载成功，可以正常使用了。

　　所有的`vfsmount`都会存放在`mount_hashtable`的哈希表中，key是`vfsmount`地址以及dentry地址计算出来的哈希值。

#### 以open系统调用为例小结vfs的基本知识

　　在继续探究`vfs_read`和`vfs_write`之前，先通过open系统调用去串连一下vfs层的4个对象，小结一下前面的内容。因为open调用仅仅涉及到vfs层，跟磁盘高速缓存以及具体文件系统的实现基本无关。

　　回忆一下前面的hello world例子，在进行文件拷贝前，先要open文件：

	fd1 = open("helloworld.in", O_RONLY);
	fd2 = open("helloworld.out", O_WRONLY);

　　这里核心的任务就是要通过传入的路径参数，最终创建出vfs的file对象。file对象确定了以后，意味着对应的inode,dentry和superblock也确定了，4大vfs对象全都准备后，可以接受读写请求了。最后返回其在内核进程的文件打开数组里的索引号给上层用户进程。具体步骤如下：

##### 路径查找

　　首先进行路径查找，调用`path_lookup()`函数。该函数主要接受两个参数，一个是路径名，一个是nameidata类型的结构体,这个结构体有一个比较重要的字段是path，主要分析这个字段，在路径查找的过程中会不断修改这个字段，最后这个字段就代表路径查找的最终结果。该字段利用`vfsmount`和`dentry`唯一确定了某个路径。

	struct path {
		struct vfsmount *mnt;
		struct dentry *dentry;
	};

1. 首先判断路径是绝对路径还是相对路径，决定用进程的root还是pwd字段去填充这个path结构体，作为起始参数。
2. 用/去划分路径，依次解析每一层路径，对于每一层路径，首先找出其目录项的dentry对象，大概率会在目录项高速缓存中命中，如果缓存中没有，则读取磁盘，然后放到缓存中，并更新path字段。
3. 检查该层目录的dentry是否是某文件系统的挂载点，如果是,则用当前path的vfsmount和dentry计算哈希值，找出mount_hashtable中的子文件系统的`vfsmount`和`dentry`，并更新path的`vfsmount`和`dentry`。
4. 直到把所有分路径都解析完成，获得了最后的path。

##### 创建file对象

　　找到了目的路径的`vfsmount`和`dentry`，inode和superblock对象也相应确定了，剩下的工作就是分配一个新的文件对象，并把相应的字段用inode，dentry和vfsmount的地址去填充。

#### vfs_read, vfs_write

　　现在，我们已经有了足够的vfs知识，可以探索一下前面hello_world程序里的vfs_read和vfs_write函数了。

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
	ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
	{
		ssize_t ret;
		if (!(file->f_mode & FMODE_READ)) 
			return -EBADF;
		if (!file->f_op || (!file->f_op->read && !file->f_op->aio_read))  
			return -EINVAL;
		if (unlikely(!access_ok(VERIFY_WRITE, buf, count)))  
			return -EFAULT;
		ret = rw_verify_area(READ, file, pos, count);
		if (ret >= 0) {
			count = ret;
			if (file->f_op->read)    
				ret = file->f_op->read(file, buf, count, pos);
			else
				ret = do_sync_read(file, buf, count, pos);
			if (ret > 0) {
				fsnotify_access(file);
				add_rchar(current, ret);
			}
			inc_syscr(current);
		}
		return ret;
	}
	ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
	{
		ssize_t ret;
		if (!(file->f_mode & FMODE_WRITE))
			return -EBADF;
		if (!file->f_op || (!file->f_op->write && !file->f_op->aio_write))
			return -EINVAL;
		if (unlikely(!access_ok(VERIFY_READ, buf, count)))
			return -EFAULT;
		ret = rw_verify_area(WRITE, file, pos, count);
		if (ret >= 0) {
			count = ret;
			file_start_write(file);
			if (file->f_op->write)
				ret = file->f_op->write(file, buf, count, pos);    
			else
				ret = do_sync_write(file, buf, count, pos);
			if (ret > 0) {
				fsnotify_modify(file);
				add_wchar(current, ret);
			}
			inc_syscw(current);
			file_end_write(file);
		}
		return ret;
	}

　　可以看到，read，write系统调用会根据文件描述符提取出file对象，这个file对象是在Open调用里已经创建好的。然后会在file对象中读取出当前的文件偏移量，读写都会从这个偏移量开始。然后把file对象，用户层buffer的地址，要读写的大小，以及文件偏移量作为参数传入`vfs_read`和`vfs_write`中。这两个函数主要是对file对象的相应文件系统的读写函数进行封装，因此，主要的逻辑就过渡到了具体文件系统上了。具体文件系统的实现逻辑各不相同，但都要以vfs的4大对象为对外的接口，然后再定义自己的数据结构与方法。

　　对于通用的磁盘文件系统，linux提供了很多基本的函数，很多文件系统的核心功能都是以这些基本函数为基础，再封装一层而已。我们就以常用的xfs文件系统为例，去简单看看它的read和write函数干了什么。

	const struct file_operations xfs_file_operations = {
		.llseek		= xfs_file_llseek,
		.read		= do_sync_read,
		.write		= do_sync_write,
		.aio_read	= xfs_file_aio_read,
		.aio_write	= xfs_file_aio_write,
		.splice_read	= xfs_file_splice_read,
		.splice_write	= xfs_file_splice_write,
		.unlocked_ioctl	= xfs_file_ioctl,
	#ifdef CONFIG_COMPAT
		.compat_ioctl	= xfs_file_compat_ioctl,
	#endif
		.mmap		= xfs_file_mmap,
		.open		= xfs_file_open,
		.release	= xfs_file_release,
		.fsync		= xfs_file_fsync,
		.fallocate	= xfs_file_fallocate,
	};

　　这是xfs的f_op的函数指针表，可以看到，它的read,write函数竟然直接用了内核提供的函数，非常偷懒！

	ssize_t do_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
	{
		struct iovec iov = { .iov_base = buf, .iov_len = len };  
		struct kiocb kiocb;
		ssize_t ret;
		init_sync_kiocb(&kiocb, filp);
		kiocb.ki_pos = *ppos;
		kiocb.ki_left = len;
		kiocb.ki_nbytes = len;
		ret = filp->f_op->aio_read(&kiocb, &iov, 1, kiocb.ki_pos);  
		if (-EIOCBQUEUED == ret)
			ret = wait_on_sync_kiocb(&kiocb);
		*ppos = kiocb.ki_pos;
		return ret;
	}
	ssize_t do_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
	{
		struct iovec iov = { .iov_base = (void __user *)buf, .iov_len = len };
		struct kiocb kiocb;
		ssize_t ret;
		init_sync_kiocb(&kiocb, filp);
		kiocb.ki_pos = *ppos;
		kiocb.ki_left = len;
		kiocb.ki_nbytes = len;
		ret = filp->f_op->aio_write(&kiocb, &iov, 1, kiocb.ki_pos);    
		if (-EIOCBQUEUED == ret)
			ret = wait_on_sync_kiocb(&kiocb);
		*ppos = kiocb.ki_pos;
		return ret;
	}

　　这两个函数实际上调用了具体文件系统的aio_read和aio_write函数，而xfs文件系统的这两个函数是自定义的，xfs_file_aio_read和xfs_file_aio_write。

　　这两个函数的代码就有点复杂了，不过我们不需要细究xfs的实现细节，我们的目的是要通过xfs文件系统去找出通用的磁盘文件系统的共性，xfs_file_aio_read和xfs_file_aio_write虽然有很多xfs自己的实现细节，但其核心功能都是建立在内核提供的通用函数上的，例如xfs_file_aio_read最终会调用generic_file_aio_read函数，而xfs_file_aio_write则最终会调用generic_perform_write函数。这些通用函数是基本上所有文件系统的核心逻辑。

　　进入到这里，就开始涉及到高速缓存这一层了。我们先立足于vfs这一层，然后预热一下高速缓存层，先直接概括一下generic_file_aio_read函数做了什么事情，非常简单：

1. 根据文件偏移量计算出在要读的内容在高速缓存中的位置。
2. 搜索高速缓存，看看要读的内容是否在高速缓存中存在，若存在则直接返回，若不存在则触发读磁盘任务。若触发读磁盘任务，则判断当前是否顺序读，尽量预读取磁盘数据到高速缓存中，减少磁盘io的次数。
3. 数据读取后，拷贝到用户态的buffer中。

　　`generic_perform_write`的逻辑则是：

1. 根据文件偏移量计算出在要写的内容在高速缓存中的位置。
2. 搜索看看要写的内容是否已在高速缓存中分配了相应的数据结构，若没有，则分配相应内存空间。
3. 从用户态的buffer拷贝数据到高速缓存中。
4. 检查高速缓存中的空间是否用得太多，如果占用过多内存则唤醒后台的写回磁盘的线程，把高速缓存的部分内容写回到磁盘上，可能会造成不定时间的写阻塞。
5. 向上层返回写的结果。

### <a name="chapter5"></a>高速缓存

信号驱动式I/O：首先我们允许Socket进行信号驱动IO,并安装一个信号处理函数，进程继续运行并不阻塞。当数据准备好时，进程会收到一个SIGIO信号，可以在信号处理函数中调用I/O操作函数处理数据。过程如下图所示：

![](https://s3.uuu.ovh/imgs/2023/04/08/04935453025bdbb8.png)

### <a name="chapter6"></a>5、异步IO模型

相对于同步IO，异步IO不是顺序执行。用户进程进行aio_read系统调用之后，无论内核数据是否准备好，都会直接返回给用户进程，然后用户态进程可以去做别的事情。等到socket数据准备好了，内核直接复制数据给进程，然后从内核向进程发送通知。IO两个阶段，进程都是非阻塞的。异步过程如下图所示：

![](https://s3.uuu.ovh/imgs/2023/04/08/2ca15f43fc8b9e37.png)

### 参考资源

* [南国倾城](https://wjqwsp.github.io/2018/12/18/linux-io%E8%BF%87%E7%A8%8B%E8%87%AA%E9%A1%B6%E5%90%91%E4%B8%8B%E5%88%86%E6%9E%90/)

* [Elvis Zhang](https://blog.shunzi.tech/post/Linux_Storage_Stack/)

转载请注明：[sizheluo的博客](https://sizheluo.github.io) » [Linux下的五种IO模型](https://sizheluo.github.io/2023/04/Linux-IO%E6%A8%A1%E5%9E%8B//)
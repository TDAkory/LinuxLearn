# Linux下的文件锁

> [Linux文件锁内核VFS层源码实现讲解](https://blog.csdn.net/SaberJYang/article/details/126335051)

- [Linux下的文件锁](#linux下的文件锁)
	- [起源](#起源)
		- [`flock`和`fcntl`概要](#flock和fcntl概要)
		- [`inode`中的锁信息](#inode中的锁信息)
		- [`file_lock`的定义](#file_lock的定义)
	- [flock](#flock)
		- [调用链](#调用链)
		- [核心实现](#核心实现)
		- [锁加入](#锁加入)
		- [锁删除](#锁删除)
		- [锁冲突检测](#锁冲突检测)
		- [锁加入阻塞队列](#锁加入阻塞队列)
		- [唤醒锁](#唤醒锁)
	- [fcntl](#fcntl)


## 起源

### `flock`和`fcntl`概要

Linux下多进程同时读写同一个文件的场景十分常见，为了保证这种场景下文件内容的一致性，Linux内核提供了2种特殊的系统调用`flock`和`lockf`。

`flock`可以实现对整个文件的加锁，而`lockf`则是另一个系统调用`fcntl`的再封装，它可以实现对文件部分字节加锁，比`flock`的粒度要细。

在**linux-kernel-v5.4**版本上，文件`/fs/locks.c`中，我么可以看到如下注释解释了两种不同类型的锁

```c
 *  Implemented two lock personalities - FL_FLOCK and FL_POSIX.
 *
 *  FL_POSIX locks are created with calls to fcntl() and lockf() through the
 *  fcntl() system call. They have the semantics described above.
 *
 *  FL_FLOCK locks are created with calls to flock(), through the flock()
 *  system call, which is new. Old C libraries implement flock() via fcntl()
 *  and will continue to use the old, broken implementation.
 *
 *  FL_FLOCK locks follow the 4.4 BSD flock() semantics. They are associated
 *  with a file pointer (filp). As a result they can be shared by a parent
 *  process and its children after a fork(). They are removed when the last
 *  file descriptor referring to the file pointer is closed (unless explicitly
 *  unlocked).
 *
 *  FL_FLOCK locks never deadlock, an existing lock is always removed before
 *  upgrading from shared to exclusive (or vice versa). When this happens
 *  any processes blocked by the current lock are woken up and allowed to
 *  run before the new lock is applied.
```

通过参考`flock`的对应shell命令，

```shell
# man flock

NAME
       flock - manage locks from shell scripts

SYNOPSIS
       flock [options] file|directory command [arguments]
       flock [options] file|directory -c command
       flock [options] number

DESCRIPTION
       This utility manages flock(2) locks from within shell scripts or from the command line.

       The first and second of the above forms wrap the lock around the execution of a command, in a manner similar to su(1) or newgrp(1).  They lock a specified file or directory, which is created (assuming appropriate
       permissions) if it does not already exist.  By default, if the lock cannot be immediately acquired, flock waits until the lock is available.

       The third form uses an open file by its file descriptor number.  See the examples below for how that can be used.
```

`fcntl`的man page

```shell
NAME
       fcntl - manipulate file descriptor

SYNOPSIS
       #include <unistd.h>
       #include <fcntl.h>

       int fcntl(int fd, int cmd, ... /* arg */ );

DESCRIPTION
       fcntl() performs one of the operations described below on the open file descriptor fd.  The operation is determined by cmd.

       fcntl()  can take an optional third argument.  Whether or not this argument is required is determined by cmd.  The required argument type is indicated in parentheses after each cmd name (in most cases, the required
       type is int, and we identify the argument using the name arg), or void is specified if the argument is not required.

       Certain of the operations below are supported only since a particular Linux kernel version.  The preferred method of checking whether the host kernel supports a particular operation is to invoke  fcntl()  with  the
       desired cmd value and then test whether the call failed with EINVAL, indicating that the kernel does not recognize this value.
```

可以看到flock的操作对象是`file|directory`，在一切都是文件的linux系统中，其对应于文件系统的inode，因此可以从inode开始分析内核代码，找到flock相关的信息。

### `inode`中的锁信息

指向磁盘上相同文件、相同类型的所有`file_lock`结构，都被收集在`inode`的`i_flctx`，其内部根据锁类型划分了三个锁链表：

```c
// /include/linux/fs.h
struct inode {
    ...
    struct file_lock_context	*i_flctx;
    ...
};

struct file_lock_context {
    spinlock_t          flc_lock;   /* protects inode rw: flc_flock, flc_posix, flc_lease */
    struct list_head    flc_flock;  // flock
    struct list_head    flc_posix;  // fcntl
    struct list_head    flc_lease;
};

// file_lock_context 的初始化接口
static struct file_lock_context *locks_get_lock_context(struct inode *inode, int type)
{
	struct file_lock_context *ctx;

	/* paired with cmpxchg() below */
	ctx = smp_load_acquire(&inode->i_flctx);
	if (likely(ctx) || type == F_UNLCK)
		goto out;

	ctx = kmem_cache_alloc(flctx_cache, GFP_KERNEL);
	if (!ctx)
		goto out;

	spin_lock_init(&ctx->flc_lock);
	INIT_LIST_HEAD(&ctx->flc_flock);
	INIT_LIST_HEAD(&ctx->flc_posix);
	INIT_LIST_HEAD(&ctx->flc_lease);

	/*
	 * Assign the pointer if it's not already assigned. If it is, then
	 * free the context we just allocated.
	 */
	if (cmpxchg(&inode->i_flctx, NULL, ctx)) {
		kmem_cache_free(flctx_cache, ctx);
		ctx = smp_load_acquire(&inode->i_flctx);
	}
out:
	trace_locks_get_lock_context(inode, type, ctx);
	return ctx;
}
```

### `file_lock`的定义

可以看到，struct `file_lock` 表示一个通用的“文件锁定”。它用于表示 POSIX 字节范围锁定、BSD (flock) 锁定和租约（leases）。需要注意的是，同一个结构体既用于表示锁定请求，也用于表示锁定本身，但是同一个对象绝不会同时用于这两种用途。

各种 i_flctx 列表是按照以下顺序排列的：
1) 锁定所有者
2) 锁定范围开始
3) 锁定范围结束

显然，最后两个标准仅适用于 POSIX 类型的锁。

```c
// /include/linux/filelock.h
/*
 * struct file_lock represents a generic "file lock". It's used to represent
 * POSIX byte range locks, BSD (flock) locks, and leases. It's important to
 * note that the same struct is used to represent both a request for a lock and
 * the lock itself, but the same object is never used for both.
 *
 * The varous i_flctx lists are ordered by:
 *
 * 1) lock owner
 * 2) lock range start
 * 3) lock range end
 *
 * Obviously, the last two criteria only matter for POSIX locks.
 */
struct file_lock {
    struct file_lock *fl_blocker;   /* The lock, that is blocking us */
    struct list_head fl_list;       /* link into file_lock_context */
    struct hlist_node fl_link;      /* node in global lists */
    struct list_head fl_blocked_requests;   /* list of requests with ->fl_blocker pointing here */
    struct list_head fl_blocked_member;     /* node in ->fl_blocker->fl_blocked_requests */
    fl_owner_t fl_owner;
    unsigned int fl_flags;          // 锁标志
    unsigned char fl_type;          // 锁类型
    unsigned int fl_pid;            // 进程拥有者的pid
    int fl_link_cpu;                /* what cpu's list is this on? */
    wait_queue_head_t fl_wait;      // 被阻塞进程的等待队列
    struct file *fl_file;           // 指向被锁的文件
    loff_t fl_start;                // 被锁区域的开始偏移
    loff_t fl_end;                  // 被锁区域的结束偏移

    struct fasync_struct *	fl_fasync; /* for lease break notifications */
    /* for lease breaks: */
    unsigned long fl_break_time;        // 租约剩余时间
    unsigned long fl_downgrade_time;

    const struct file_lock_operations *fl_ops;      /* Callbacks for filesystems */
    const struct lock_manager_operations *fl_lmops; /* Callbacks for lockmanagers */
    union {
        struct nfs_lock_info    nfs_fl;
        struct nfs4_lock_info   nfs4_fl;
        struct {
            struct list_head link;  /* link in AFS vnode's pending_locks list */
            int state;              /* state of grant or error if -ve */
            unsigned int    debug_id;
        } afs;
        struct {
            struct inode *inode;
        } ceph;
    } fl_u;
} __randomize_layout;
```

锁的类型差异主要体现在 `fl_flags` 和 `fl_type` 上，在处理流程中常用这两个标志来区分逻辑分支：

`fl_flags`的定义

```c
// /include/linux/fs.h
#define FL_POSIX	1
#define FL_FLOCK	2
#define FL_DELEG	4	/* NFSv4 delegation */
#define FL_ACCESS	8	/* not trying to lock, just looking */
#define FL_EXISTS	16	/* when unlocking, test for existence */
#define FL_LEASE	32	/* lease held on this file */
#define FL_CLOSE	64	/* unlock on close */
#define FL_SLEEP	128	/* A blocking lock */
#define FL_DOWNGRADE_PENDING	256 /* Lease is being downgraded */
#define FL_UNLOCK_PENDING	512 /* Lease is being broken */
#define FL_OFDLCK	1024	/* lock is "owned" by struct file */
#define FL_LAYOUT	2048	/* outstanding pNFS layout */
```

`fl_type`的定义

```c
// /include/uapi/asm-generic/fcntl.h
/* for posix fcntl() and lockf() */
#ifndef F_RDLCK
#define F_RDLCK		0
#define F_WRLCK		1
#define F_UNLCK		2
#endif

/* operations for bsd flock(), also used by the kernel implementation */
#define LOCK_SH		1	/* shared lock */
#define LOCK_EX		2	/* exclusive lock */
#define LOCK_NB		4	/* or'd with one of the above to prevent blocking */
#define LOCK_UN		8	/* remove lock */

#define LOCK_MAND	32	/* This is a mandatory flock ... */
#define LOCK_READ	64	/* which allows concurrent read operations */
#define LOCK_WRITE	128	/* which allows concurrent write operations */
#define LOCK_RW		192	/* which allows concurrent read & write ops */
```

在 `/fs/locks.c` 中，还定义了一些全局结构:

* file_lock_list 仅用来展示 /proc/locks 的文件内容，每个cpu持有一个
* blocked_hash 记录FL_POSIX类型的所有文件的阻塞锁，用于死锁检查

```c
/*
 * The global file_lock_list is only used for displaying /proc/locks, so we
 * keep a list on each CPU, with each list protected by its own spinlock.
 * Global serialization is done using file_rwsem.
 *
 * Note that alterations to the list also require that the relevant flc_lock is
 * held.
 */
struct file_lock_list_struct {
	spinlock_t		lock;
	struct hlist_head	hlist;
};
static DEFINE_PER_CPU(struct file_lock_list_struct, file_lock_list);
DEFINE_STATIC_PERCPU_RWSEM(file_rwsem);

/*
 * The blocked_hash is used to find POSIX lock loops for deadlock detection.
 * It is protected by blocked_lock_lock.
 *
 * We hash locks by lockowner in order to optimize searching for the lock a
 * particular lockowner is waiting on.
 *
 * FIXME: make this value scale via some heuristic? We generally will want more
 * buckets when we have more lockowners holding locks, but that's a little
 * difficult to determine without knowing what the workload will look like.
 */
#define BLOCKED_HASH_BITS	7
static DEFINE_HASHTABLE(blocked_hash, BLOCKED_HASH_BITS);
```

当不同进程打开同一个文件时（”open()”系统调用），尽管每个进程都会单独实例化一个”struct file”对象，但他们都指向同一个inode。

> 打开文件的操作流 do_sys_open --> do_sys_openat2 --> 通过`do_filp_open`获取一个struct file对象 --> struct inode
>
> 关于struct file如何关联到 struct inode，下次展开

## flock

FL_FLOCK锁，总是与一个文件对象相关联，因此由一个打开该文件的进程（或共享同一个文件描述符的子进程）来维护。只要fd映射的file对象不一样，就认为是不同的拥有者，而不关心进程是否相同。对于相同的文件，只要调用open，就会生成一个新的file对象。这也就能解释FL_FLOCK如下语义了：

- fork产生的子进程可以继承父进程所设置的锁。

- 通过dup或者fork产生的两个fd，都可以加锁而不会产生死锁，这两个fd都可以操作这把锁（例如通过一个fd加锁，通过另一个fd可以释放锁），但是上锁过程中关闭其中一个fd，并不会释放锁（因为file对象并没有释放），只有关闭所有复制出的fd，锁才会释放。

- 使用open两次打开同一个文件，得到的两个fd是独立的（因为底层对应两个file对象），通过其中一个加锁，通过另一个无法解锁。

### 调用链

shell命令中的`flock`，和`flock()`函数，最终都会调用到`sys_flock()`:

从这里的注释可以看出，`sys_flock()`对一个打开的文件描述符应用类似于FL_FLOCK风格的锁定。参数 @cmd 可以是以下之一：

- `LOCK_SH` -- 申请一个共享锁定（shared lock）。
- `LOCK_EX` -- 申请一个独占锁定（exclusive lock）。
- `LOCK_UN` -- 移除一个已存在的锁定。
- `LOCK_MAND` -- 一个'强制的'flock，用于模拟Windows的共享模式（Share Modes）。`LOCK_MAND` 可以与 `LOCK_READ` 或 `LOCK_WRITE` 结合使用，以便分别允许其他进程进行读写访问。

```c
/**
 *	sys_flock: - flock() system call.
 *	@fd: the file descriptor to lock.
 *	@cmd: the type of lock to apply.
 *
 *	Apply a %FL_FLOCK style lock to an open file descriptor.
 *	The @cmd can be one of:
 *
 *	- %LOCK_SH -- a shared lock.
 *	- %LOCK_EX -- an exclusive lock.
 *	- %LOCK_UN -- remove an existing lock.
 *	- %LOCK_MAND -- a 'mandatory' flock.
 *	  This exists to emulate Windows Share Modes.
 *
 *	%LOCK_MAND can be combined with %LOCK_READ or %LOCK_WRITE to allow other
 *	processes read and write access respectively.
 */
SYSCALL_DEFINE2(flock, unsigned int, fd, unsigned int, cmd)
{
	struct fd f = fdget(fd);	// 获取文件描述符对应的struct file对象，同时引用计数+1
	struct file_lock *lock;
	int can_sleep, unlock;
	int error;

	error = -EBADF;
	if (!f.file)	// 检测f.file是否有效
		goto out;

	can_sleep = !(cmd & LOCK_NB);	// non-blocking 则不支持等待
	cmd &= ~LOCK_NB;
	unlock = (cmd == LOCK_UN);		// 是否是解锁

	if (!unlock && !(cmd & LOCK_MAND) &&
	    !(f.file->f_mode & (FMODE_READ|FMODE_WRITE)))	// 检查file的读写权限
		goto out_putf;

	lock = flock_make_lock(f.file, cmd, NULL);	// 获取一个file_lock对象，并初始化
	if (IS_ERR(lock)) {
		error = PTR_ERR(lock);
		goto out_putf;
	}

	if (can_sleep)
		lock->fl_flags |= FL_SLEEP;		// blocking 则设置 FL_SLEEP 标志

	error = security_file_lock(f.file, lock->fl_type);
	if (error)
		goto out_free;

	if (f.file->f_op->flock)		// 如果文件系统支持flock，则调用文件系统的flock接口
		error = f.file->f_op->flock(f.file,
					  (can_sleep) ? F_SETLKW : F_SETLK,
					  lock);
	else
		error = locks_lock_file_wait(f.file, lock);

 out_free:
	locks_free_lock(lock);

 out_putf:
	fdput(f);	// 文件描述符引用计数-1
 out:
	return error;
}
```

`sys_flock()`的核心逻辑是调用`flock_make_lock()`函数，该函数会根据cmd参数构造一个`file_lock`对象，然后调用`locks_lock_file_wait()` --> `locks_lock_inode_wait()` --> `flock_lock_inode_wait()`

```c
static inline int locks_lock_file_wait(struct file *filp, struct file_lock *fl)
{
	return locks_lock_inode_wait(locks_inode(filp), fl);
}
```

其中`locks_inode`是一个宏定义，最终展开如下，直接从`struct file`中获取了`struct inode`对象，因此锁逻辑直接在`inode`上继续进行

```c
static inline struct inode *file_inode(const struct file *f)
{
	return f->f_inode;
}
```

### 核心实现

`locks_lock_file_wait()` --> `locks_lock_inode_wait()` --> `flock_lock_inode_wait()`，最终在inode上加锁的核心逻辑，在`flock_lock_inode_wait()`中

```c
// /fs/locks.c
/**
 * locks_lock_inode_wait - Apply a lock to an inode
 * @inode: inode of the file to apply to
 * @fl: The lock to be applied
 *
 * Apply a POSIX or FLOCK style lock request to an inode.
 */
int locks_lock_inode_wait(struct inode *inode, struct file_lock *fl)
{
	int res = 0;
	switch (fl->fl_flags & (FL_POSIX|FL_FLOCK)) {
		case FL_POSIX:
			res = posix_lock_inode_wait(inode, fl);
			break;
		case FL_FLOCK:
			res = flock_lock_inode_wait(inode, fl);
			break;
		default:
			BUG();
	}
	return res;
}
EXPORT_SYMBOL(locks_lock_inode_wait);

/**
 * flock_lock_inode_wait - Apply a FLOCK-style lock to a file
 * @inode: inode of the file to apply to
 * @fl: The lock to be applied
 *
 * Apply a FLOCK style lock request to an inode.
 */
static int flock_lock_inode_wait(struct inode *inode, struct file_lock *fl)
{
	int error;
	might_sleep();
	for (;;) {
		error = flock_lock_inode(inode, fl);
		if (error != FILE_LOCK_DEFERRED)
			break;
		error = wait_event_interruptible(fl->fl_wait, !fl->fl_blocker);
		if (error)
			break;
	}
	locks_delete_block(fl);
	return error;
}

/* Try to create a FLOCK lock on filp. We always insert new FLOCK locks
 * after any leases, but before any posix locks.
 *
 * Note that if called with an FL_EXISTS argument, the caller may determine
 * whether or not a lock was successfully freed by testing the return
 * value for -ENOENT.
 */
static int flock_lock_inode(struct inode *inode, struct file_lock *request)
{
	struct file_lock *new_fl = NULL;
	struct file_lock *fl;
	struct file_lock_context *ctx;
	int error = 0;
	bool found = false;
	LIST_HEAD(dispose);

	ctx = locks_get_lock_context(inode, request->fl_type);  // 获取锁上下文
	if (!ctx) {
		if (request->fl_type != F_UNLCK)
			return -ENOMEM;
		return (request->fl_flags & FL_EXISTS) ? -ENOENT : 0;
	}

	if (!(request->fl_flags & FL_ACCESS) && (request->fl_type != F_UNLCK)) {
		new_fl = locks_alloc_lock();    // 如果操作类型不是 检查 or 释放 锁，那么申请一个新的锁实例
		if (!new_fl)
			return -ENOMEM;
	}

	percpu_down_read(&file_rwsem);
	spin_lock(&ctx->flc_lock);
	if (request->fl_flags & FL_ACCESS)
		goto find_conflict;             // 如果仅仅是检查锁，那么跳到 conflict 

	list_for_each_entry(fl, &ctx->flc_flock, fl_list) {
		if (request->fl_file != fl->fl_file)    // 文件不同直接跳过
			continue;   
		if (request->fl_type == fl->fl_type)    // 文件存在 且 锁类型相同，已加锁退出？
			goto out;
		found = true;                           // 文件存在 且 锁操作的类型不同，继续
		locks_delete_lock_ctx(fl, &dispose);    // 找到的情况下，先释放锁
		break;                                  // 跳出对list的遍历
	}

	if (request->fl_type == F_UNLCK) {
		if ((request->fl_flags & FL_EXISTS) && !found)
			error = -ENOENT;
		goto out;
	}

find_conflict:
	// 用request和当前链表上的每个fl执行冲突检测，若冲突，则视request的参数决定是立即返回，还是加入对应fl的阻塞链表中
	list_for_each_entry(fl, &ctx->flc_flock, fl_list) {
		if (!flock_locks_conflict(request, fl)) // 冲突检测，主要是检查 fl 是否阻塞了 request，在flock的语义下，相同的文件不会造成冲突
			continue;                   // 未冲突的情况下继续遍历
		error = -EAGAIN;                // 检测到冲突则置error
		if (!(request->fl_flags & FL_SLEEP))    // 如果请求的锁不支持sleep等待，则跳转 out 处理
			goto out;
		error = FILE_LOCK_DEFERRED;             // 冲突 且 请求所支持阻塞，修正 error
		locks_insert_block(fl, request, flock_locks_conflict);  // Insert waiter into blocker's block list.  __locks_insert_block()
		goto out;
	}
	if (request->fl_flags & FL_ACCESS)  // 完成遍历后，若请求类型仅是检查，也跳转 out 处理
		goto out;
	// 走到这里，意味着当前inode上的flock，与入参不存在冲突，可以加锁
	locks_copy_lock(new_fl, request);   // 将锁相关的元信息复制到 new_fl 中
	locks_move_blocks(new_fl, request); // 将锁相关的阻塞信息复制到 new_fl 中
	// 把新的锁加入到ctx的flc_flock列表中，同时将锁加入到本CPU的锁列表中
    locks_insert_lock_ctx(new_fl, &ctx->flc_flock); 
	new_fl = NULL;
	error = 0;

out:
	spin_unlock(&ctx->flc_lock);
	percpu_up_read(&file_rwsem);
	if (new_fl)
		locks_free_lock(new_fl);	// 如果加锁失败，则释放预先申请的锁内存
	locks_dispose_list(&dispose);	// 释放dispose链表中的锁内存
	trace_flock_lock_inode(inode, request, error);
	return error;
}
```

“for_each_lock()”会去遍历”inode->i_flock”这个链表，那么为啥这里要搞个链表来存文件锁呢？

原因是因为flock归根到底还是个建议锁，它不要求进程一定要遵守，当一个进程对某个文件使用了文件锁，但是另一个进程却压根不去检查该文件是否已经存在文件锁，霸道地直接读写文件内容，事实上内核并不会阻止。所以，flock有效的前提是大家都遵守同样的锁规则，在读写文件前都需要提前去检查一下是否某个进程还持有着文件锁。当然，为了功能性考虑，flock还是支持LOCK_SH（共享锁）和LOCK_EX（排他锁）多种模式的，于是，用一个链表来存储不同进程的共享锁自然很有必要。

### 锁加入

### 锁删除

在上述函数中初次遍历inode上的锁链表时，如果找到了一个锁节点，其fl_file文件对象相同，但fl_type锁类型不同的时候，会将当前inode上的锁先释放掉：

1. **这也能解释在后续的锁冲突检测中，为什么看到fl_file文件对象相同时，认为冲突不存在，因为旧锁已经被释放掉了**
2. **这里可以认为flock在同一个fl_file对象上是可重入的，并且是非自旋的，这种方式下重复加锁，在inode的锁链表上，总是只有一个锁生效**

删除函数的逻辑是非常直观的，先从inode上的锁链表中删除，加入到dispose链表中，如果没有dispose链表，则立即释放；dispose链表则会由外部函数最终完成清理。

```c
static void
locks_delete_lock_ctx(struct file_lock *fl, struct list_head *dispose)
{
	locks_unlink_lock_ctx(fl);
	if (dispose)
		list_add(&fl->fl_list, dispose);
	else
		locks_free_lock(fl);
}
```

### 锁冲突检测

FL_FLOCK锁没有用到fl_owner字段，因为内核判断FL_FLOCK类型锁的拥有者，是通过fl_file字段即打开的文件对象区分的。

`locks_conflict`中的阻塞判断也是直观的，只要当前或入参有任一个是写锁，就认为是存在阻塞。

```c
/* Determine if lock sys_fl blocks lock caller_fl. FLOCK specific
 * checking before calling the locks_conflict().
 */
static bool flock_locks_conflict(struct file_lock *caller_fl,
				 struct file_lock *sys_fl)
{
	/* FLOCK locks referring to the same filp do not conflict with
	 * each other.
	 */
	if (caller_fl->fl_file == sys_fl->fl_file)
		return false;
	if ((caller_fl->fl_type & LOCK_MAND) || (sys_fl->fl_type & LOCK_MAND))
		return false;

	return locks_conflict(caller_fl, sys_fl);
}

/* Determine if lock sys_fl blocks lock caller_fl. Common functionality
 * checks for shared/exclusive status of overlapping locks.
 */
static bool locks_conflict(struct file_lock *caller_fl,
			   struct file_lock *sys_fl)
{
	if (sys_fl->fl_type == F_WRLCK)
		return true;
	if (caller_fl->fl_type == F_WRLCK)
		return true;
	return false;
}
```

### 锁加入阻塞队列

在上面的冲突检测中，检测到冲突、且入参支持超时等待的request，会被加入到对应fl的阻塞队列中

```c
static void locks_insert_block(struct file_lock *blocker,
			       struct file_lock *waiter,
			       bool conflict(struct file_lock *,
					     struct file_lock *))
{
	spin_lock(&blocked_lock_lock);
	__locks_insert_block(blocker, waiter, conflict);
	spin_unlock(&blocked_lock_lock);
}

/* Insert waiter into blocker's block list.
 * We use a circular list so that processes can be easily woken up in
 * the order they blocked. The documentation doesn't require this but
 * it seems like the reasonable thing to do.
 *
 * Must be called with both the flc_lock and blocked_lock_lock held. The
 * fl_blocked_requests list itself is protected by the blocked_lock_lock,
 * but by ensuring that the flc_lock is also held on insertions we can avoid
 * taking the blocked_lock_lock in some cases when we see that the
 * fl_blocked_requests list is empty.
 *
 * Rather than just adding to the list, we check for conflicts with any existing
 * waiters, and add beneath any waiter that blocks the new waiter.
 * Thus wakeups don't happen until needed.
 */
static void __locks_insert_block(struct file_lock *blocker,
				 struct file_lock *waiter,
				 bool conflict(struct file_lock *,
					       struct file_lock *))
{
	struct file_lock *fl;
	BUG_ON(!list_empty(&waiter->fl_blocked_member));

new_blocker:
	// 一个基于阻塞队列的深度优先遍历，从入参 blocker 的阻塞队列开始，查找与入参 waiter 也存在冲突的队列上waiter，并将这个waiter更新为blocker
	// 最终，新的 waiter 会被插入到那个阻塞它的请求之后。这样做的目的是维护等待队列的顺序，确保按照特定的规则（可能是先来先服务）来授予锁。
	list_for_each_entry(fl, &blocker->fl_blocked_requests, fl_blocked_member) 
		if (conflict(fl, waiter)) {
			blocker =  fl;
			goto new_blocker;
		}
	waiter->fl_blocker = blocker;	// 更新waiter的fl_blocker属性
	list_add_tail(&waiter->fl_blocked_member, &blocker->fl_blocked_requests);	// 将waiter加入到blocker的阻塞队列中
	if (IS_POSIX(blocker) && !IS_OFDLCK(blocker))
		// 上述条件下，将waiter加入到block_hash中
		locks_insert_global_blocked(waiter);

	/* The requests in waiter->fl_blocked are known to conflict with
	 * waiter, but might not conflict with blocker, or the requests
	 * and lock which block it.  So they all need to be woken.
	 */
	__locks_wake_up_blocks(waiter);		// 尝试唤醒 waiter->fl_blocked_requests 上的锁节点
}
```

### 唤醒锁

## fcntl

fcntl的系统调用函数

```c
SYSCALL_DEFINE3(fcntl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
{	
	struct fd f = fdget_raw(fd);
	long err = -EBADF;

	if (!f.file)
		goto out;

	if (unlikely(f.file->f_mode & FMODE_PATH)) {
		if (!check_fcntl_cmd(cmd))
			goto out1;
	}

	err = security_file_fcntl(f.file, cmd, arg);
	if (!err)
		err = do_fcntl(fd, cmd, arg, f.file);

out1:
 	fdput(f);
out:
	return err;
}
```

其主要调用路径是 `fcntl()` --> `do_fcntl()`，`do_fcntl`会根据cmd参数调用不同的函数，比如：`F_GETLK`用来查询文件的锁信息，`F_SETLK`和`F_SETLKW`用来对文件的某些字节执行加锁操作。在加锁的路径上，会调用到 `fcntl_setlk()`。在`fcntl_setlk()`，有两个关键的调用，分别是 `flock_to_posix_lock` 和 `do_lock_file_wait`。

`flock_to_posix_lock()` --> `flock64_to_posix_lock()`的逻辑比较简单，其针对文件字节加锁的区间计算，并做了锁类型的设置。

```c
/* Verify a "struct flock" and copy it to a "struct file_lock" as a POSIX
 * style lock.
 */
static int flock_to_posix_lock(struct file *filp, struct file_lock *fl,
			       struct flock *l)
{
	struct flock64 ll = {
		.l_type = l->l_type,
		.l_whence = l->l_whence,
		.l_start = l->l_start,
		.l_len = l->l_len,
	};

	return flock64_to_posix_lock(filp, fl, &ll);
}

static int flock64_to_posix_lock(struct file *filp, struct file_lock *fl,
				 struct flock64 *l)
{
	switch (l->l_whence) {
	case SEEK_SET:
		fl->fl_start = 0;
		break;
	case SEEK_CUR:
		fl->fl_start = filp->f_pos;
		break;
	case SEEK_END:
		fl->fl_start = i_size_read(file_inode(filp));
		break;
	default:
		return -EINVAL;
	}
	if (l->l_start > OFFSET_MAX - fl->fl_start)
		return -EOVERFLOW;
	fl->fl_start += l->l_start;
	if (fl->fl_start < 0)
		return -EINVAL;

	/* POSIX-1996 leaves the case l->l_len < 0 undefined;
	   POSIX-2001 defines it. */
	if (l->l_len > 0) {
		if (l->l_len - 1 > OFFSET_MAX - fl->fl_start)
			return -EOVERFLOW;
		fl->fl_end = fl->fl_start + l->l_len - 1;

	} else if (l->l_len < 0) {
		if (fl->fl_start + l->l_len < 0)
			return -EINVAL;
		fl->fl_end = fl->fl_start - 1;
		fl->fl_start += l->l_len;
	} else
		fl->fl_end = OFFSET_MAX;

	fl->fl_owner = current->files;
	fl->fl_pid = current->tgid;
	fl->fl_file = filp;
	fl->fl_flags = FL_POSIX;
	fl->fl_ops = NULL;
	fl->fl_lmops = NULL;

	return assign_type(fl, l->l_type);
}
```

`do_lock_file_wait()` --> `vfs_lock_file()` --> `posix_lock_file()` --> `posix_lock_inode()`。其加锁逻辑与`flock`是类似的，都是在`inode->i_flock`这个链表上增删改查，但是比`flock`复杂的地方在于`fcntl`需要考虑记录锁区间的交叉等问题。

```c
/**
 * vfs_lock_file - file byte range lock
 * @filp: The file to apply the lock to
 * @cmd: type of locking operation (F_SETLK, F_GETLK, etc.)
 * @fl: The lock to be applied
 * @conf: Place to return a copy of the conflicting lock, if found.
 *
 * A caller that doesn't care about the conflicting lock may pass NULL
 * as the final argument.
 *
 * If the filesystem defines a private ->lock() method, then @conf will
 * be left unchanged; so a caller that cares should initialize it to
 * some acceptable default.
 *
 * To avoid blocking kernel daemons, such as lockd, that need to acquire POSIX
 * locks, the ->lock() interface may return asynchronously, before the lock has
 * been granted or denied by the underlying filesystem, if (and only if)
 * lm_grant is set. Callers expecting ->lock() to return asynchronously
 * will only use F_SETLK, not F_SETLKW; they will set FL_SLEEP if (and only if)
 * the request is for a blocking lock. When ->lock() does return asynchronously,
 * it must return FILE_LOCK_DEFERRED, and call ->lm_grant() when the lock
 * request completes.
 * If the request is for non-blocking lock the file system should return
 * FILE_LOCK_DEFERRED then try to get the lock and call the callback routine
 * with the result. If the request timed out the callback routine will return a
 * nonzero return code and the file system should release the lock. The file
 * system is also responsible to keep a corresponding posix lock when it
 * grants a lock so the VFS can find out which locks are locally held and do
 * the correct lock cleanup when required.
 * The underlying filesystem must not drop the kernel lock or call
 * ->lm_grant() before returning to the caller with a FILE_LOCK_DEFERRED
 * return code.
 */
int vfs_lock_file(struct file *filp, unsigned int cmd, struct file_lock *fl, struct file_lock *conf)
{
	if (filp->f_op->lock)
		return filp->f_op->lock(filp, cmd, fl);
	else
		return posix_lock_file(filp, fl, conf);
}
EXPORT_SYMBOL_GPL(vfs_lock_file);
```
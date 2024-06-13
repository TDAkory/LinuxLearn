# [io uring](https://en.wikipedia.org/wiki/Io_uring)

- [Lord of the io_uring](https://unixism.net/loti/async_intro.html)
- [liburing](https://github.com/axboe/liburing)

## 思考模型

> [The Low-level io_uring Interface](https://unixism.net/loti/low_level.html#low-level)

- 存在两个独立的`ring buffer`，一个是用于接收请求（submission queue，SQ），一个是用于通知已完成的请求（completion queue，CQ）
- `SQ` 和 `CQ` 在内核和用户态之间共享。它们的地址和大小是通过 `io_uring_setup` 函数传递给内核的，然后通过`mmap`将其映射到用户空间。
- 将需要完成的操作（例如读取或写入文件，接受客户端连接等），封装为提交队列条目（Submission Queue Entry，简称 SQE），并将其添加到`SQ`的尾部，通过这种方式将请求发送到内核。
- 通过 `io_uring_enter()` 系统调用来告诉内核您已将 `SQE` 添加到`SQ`，也可以在执行系统调用之前添加多个 `SQE`。
  - 内核还可以自动轮询提交队列的新条目，无需调用 `io_uring_enter()` [Submission Queue Polling](https://unixism.net/loti/tutorial/sq_poll.html#sq-poll)
- 可选地，`io_uring_enter()` 还可以等待内核处理一定数量的请求后才返回，这样可以立即从`CQ`读取结果。
- 内核处理提交的请求，并将 `CQE` 添加到`CQ`环形缓冲区的尾部。
- 用户从 `CQ` 环形缓冲区的头部读取 `CQE`。每个 `SQE` 对应一个 `CQE`，它包含特定请求的状态。
- 用户根据需要继续添加 `SQE` 并收获 `CQE`。

## 深入源码

> /include/uapi/linux/io_uring.h

### CQE

```c
/*
 * IO completion data structure (Completion Queue Entry)
 */
struct io_uring_cqe {
	__u64	user_data;	/* sqe->data submission passed back */
	__s32	res;		/* result code for this event */
	__u32	flags;
};
```

user_data是从SQE原封不动传递到CQE的，这样做的原因是，kernel生成的CQE并不需要按照SQE的顺序保持一一对应（比如不同的SQE实际执行的耗时可能不同，为了尽快获得返回结果，kernel不需要严格按序生成CQE），所以需要一个字段来保存SQE的索引。

上述仅是io_uring的默认行为，我们可以使用SQE排序强制对某些操作进行排序，将它们链接起来，这主要是通过设置SQE的flags字段来实现的：IOSQE_IO_LINK。

### SQE

```c
/*
 * IO submission data structure (Submission Queue Entry)
 */
struct io_uring_sqe {
	__u8	opcode;		/* type of operation for this sqe */
	__u8	flags;		/* IOSQE_ flags */
	__u16	ioprio;		/* ioprio for the request */
	__s32	fd;		    /* file descriptor to do IO on */
	__u64	off;		/* offset into file */
	__u64	addr;		/* pointer to buffer or iovecs */
	__u32	len;		/* buffer size or number of iovecs */
	union {
		__kernel_rwf_t	rw_flags;
		__u32		fsync_flags;
		__u16		poll_events;
		__u32		sync_range_flags;
		__u32		msg_flags;
		__u32		timeout_flags;
	};
	__u64	user_data;	/* data to be passed back at completion time */
	union {
		__u16	buf_index;	/* index into fixed buffers, if used */
		__u64	__pad2[3];
	};
};
```

opcode用来指定操作类型，linux定义了如下的可选值：

```c
#define IORING_OP_NOP		0
#define IORING_OP_READV		1
#define IORING_OP_WRITEV	2
#define IORING_OP_FSYNC		3
#define IORING_OP_READ_FIXED	4
#define IORING_OP_WRITE_FIXED	5
#define IORING_OP_POLL_ADD	6
#define IORING_OP_POLL_REMOVE	7
#define IORING_OP_SYNC_FILE_RANGE	8
#define IORING_OP_SENDMSG	9
#define IORING_OP_RECVMSG	10
#define IORING_OP_TIMEOUT	11
```
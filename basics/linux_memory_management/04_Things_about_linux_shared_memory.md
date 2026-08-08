# Things about Linux 共享内存体系

> 本文整理 Linux 共享内存的完整体系：`System V shm`、`POSIX shm_open`、`tmpfs/shmem`、`memfd_create`、`hugetlbfs`。重点是分清 API、底层 backing、生命周期、路径/FD 语义、是否可 swap、是否支持大页，以及它们如何与 `THP`、`hugetlbfs` 组合成高性能 IPC 和大内存服务。

## 0. 一句话总览

共享内存的本质是：**多个进程把同一批物理页映射到各自虚拟地址空间中。**

```text
Process A virtual address        Process B virtual address
  0x7f00_0000 ----+                 0x6a00_0000 ----+
                  |                                 |
                  v                                 v
              same physical pages / same file-backed pages
```

因此共享内存的价值是：

- 进程间传输大块数据时避免用户态来回复制；
- 多进程共享缓存、队列、ring buffer、模型参数或数据页；
- 通过 `mmap` 把“文件对象”变成“内存对象”；
- 和大页结合后降低 TLB 压力。

但共享内存只解决“数据页共享”，不自动解决：

- 并发同步；
- 内存一致性协议；
- 对象生命周期；
- ABI/layout 兼容；
- 崩溃恢复；
- 权限隔离。

## 1. 先分清 API 和 backing

Linux 共享内存经常被混为一谈。更准确的视角是：API 只是入口，底层 backing 决定内存行为。

```text
API / object name
  System V shm id
  POSIX shm name
  tmpfs path
  memfd fd
  hugetlbfs path
        |
        v
file or pseudo-file object
        |
        v
mmap into processes
        |
        v
shared physical pages
```

| 机制 | 入口 | backing | 是否有路径 | 典型用途 |
| --- | --- | --- | --- | --- |
| `System V shm` | `shmget/shmat` | System V IPC 对象 | 无普通文件路径 | 老系统、遗留 IPC |
| `POSIX shm_open` | `/dev/shm/<name>` | tmpfs/shmem | 固定在 `/dev/shm` | 标准跨进程共享内存 |
| tmpfs 文件 | 任意 tmpfs 挂载点 | tmpfs/shmem | 有路径 | 自定义容量/权限/策略共享内存 |
| `memfd_create` | anonymous fd | shmem-backed file | 无路径 | 容器、RPC、FD 传递、临时共享对象 |
| `hugetlbfs` | hugetlbfs 文件 | hugetlb page pool | 有挂载路径 | 低延迟大页共享内存 |

注意：`POSIX shm`、tmpfs 文件、`memfd` 都和 shmem/tmpfs 体系关系密切；但 `hugetlbfs` 使用的是 huge page pool，不等同于普通 shmem。

## 2. 共享内存的基础数据路径

以文件型共享内存为例：

```text
create shared object
  shm_open / open(tmpfs file) / memfd_create
        |
        v
ftruncate to target size
        |
        v
mmap MAP_SHARED in process A
mmap MAP_SHARED in process B
        |
        v
same underlying pages
```

写入路径：

```text
Process A store
  -> CPU cache
  -> shared physical page
  -> Process B load same cache-coherent memory
```

这里没有内核在两进程之间复制数据。内核参与的是对象创建、映射建立、缺页、权限检查和回收。真正的读写发生在用户态 load/store 上。

同步必须另做：

```text
shared data page
  + futex / pthread_mutexattr(PTHREAD_PROCESS_SHARED)
  + eventfd
  + pipe/socket notification
  + atomics + memory ordering
```

## 3. `System V shm`

### 3.1 API

`System V` 共享内存使用一组传统 IPC API：

```c
int shmget(key_t key, size_t size, int shmflg);
void *shmat(int shmid, const void *shmaddr, int shmflg);
int shmdt(const void *shmaddr);
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

典型流程：

```text
shmget(key, size, IPC_CREAT)
        |
        v
shmat(shmid)
        |
        v
read/write shared memory
        |
        v
shmdt
        |
        v
shmctl(IPC_RMID)
```

### 3.2 特点

- 通过 `key` 和 `shmid` 管理；
- 不需要普通文件路径；
- 生命周期可以独立于创建进程；
- 需要显式 `IPC_RMID` 删除；
- 可通过 `ipcs` / `ipcrm` 查看和清理；
- 常见于老项目、数据库或传统 UNIX IPC。

### 3.3 优缺点

优点：

- 历史悠久，系统支持广；
- 不依赖挂载路径；
- 对遗留系统兼容性好。

缺点：

- 命名和生命周期不直观；
- 残留对象需要额外清理；
- 和文件描述符生态结合不如 `memfd`；
- 权限、命名空间和容器隔离语义更复杂。

现代新项目通常优先考虑 `memfd`、POSIX shm 或 tmpfs 文件。

## 4. POSIX shared memory：`shm_open`

### 4.1 API

```c
int fd = shm_open("/name", O_CREAT | O_RDWR, 0600);
ftruncate(fd, size);
void *p = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

释放：

```c
munmap(p, size);
close(fd);
shm_unlink("/name");
```

### 4.2 `/dev/shm` 语义

POSIX shm 对象通常出现在 `/dev/shm`：

```text
shm_open("/foo")
        |
        v
/dev/shm/foo
        |
        v
tmpfs/shmem file
        |
        v
mmap MAP_SHARED
```

它本质上是一个 tmpfs 文件对象，只是通过 POSIX shm API 创建和命名。

### 4.3 容量限制

`/dev/shm` 是 tmpfs，容量受 mount size 限制。容器环境里 `/dev/shm` 默认可能很小。

典型问题：

```text
large POSIX shm allocation
        |
        v
ftruncate succeeds maybe
        |
        v
page fault / write
        |
        v
SIGBUS or ENOSPC-like failure if tmpfs limit exhausted
```

工程上要检查：

```bash
df -h /dev/shm
mount | grep /dev/shm
```

容器里常见参数：

```bash
docker run --shm-size=...
```

### 4.4 适用场景

适合：

- 简单跨进程共享内存；
- 多进程服务之间共享 ring buffer；
- 需要有名字、可调试、可观察的共享对象；
- 兼容 POSIX API 的项目。

不适合：

- 不希望暴露路径；
- 强隔离容器 IPC；
- 需要精确控制 mount 点、THP 策略、容量和权限的场景。

## 5. tmpfs / shmem：通用文件型共享内存

### 5.1 `tmpfs` 是什么

`tmpfs` 是内存文件系统，文件内容存储在内存页中，必要时可以被 swap。常见挂载点：

```text
/dev/shm
/run
/tmp              # 取决于发行版配置
/run/user/<uid>
custom tmpfs mount
```

使用方式：

```bash
mount -t tmpfs -o size=16G tmpfs /mnt/myshm
```

然后： 

```text
open /mnt/myshm/buffer
ftruncate
mmap MAP_SHARED
```

### 5.2 `shmem` 与 tmpfs 的关系

内核里 shmem 是支持 tmpfs 和匿名共享内存的一套机制。用户态常见表现是 tmpfs 文件、POSIX shm、memfd 等。

可以这样理解：

```text
tmpfs path file
POSIX shm object
memfd anonymous file
        |
        v
shmem-backed file object
        |
        v
page cache-like memory pages
```

这里的“page cache-like”表示它有文件对象和页缓存式管理，但不一定对应磁盘文件。

### 5.3 tmpfs 的优势

- 路径可控；
- 容量可控；
- 权限和 mount namespace 可控；
- 可结合 cgroup；
- 可用于普通文件 API；
- 支持 `mmap MAP_SHARED`；
- 可受 shmem THP 策略影响。

### 5.4 tmpfs 的风险

- 容量满时可能触发运行时错误；
- 可 swap 时延迟可能变差；
- 需要管理文件生命周期；
- 路径暴露带来权限和清理问题；
- 容器内 mount namespace 可能和宿主不同。

## 6. `memfd_create`：匿名文件描述符共享内存

### 6.1 核心语义

`memfd_create` 创建一个匿名内存文件，返回 fd。它没有普通路径，适合通过 fd 传递共享。

```c
int fd = memfd_create("buffer", MFD_CLOEXEC);
ftruncate(fd, size);
void *p = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

把 fd 传给另一个进程：

```text
Unix domain socket
  sendmsg + SCM_RIGHTS
        |
        v
receiver gets fd
        |
        v
mmap same memfd
```

### 6.2 为什么适合现代 IPC

`memfd` 的优势是 fd 语义：

- 无路径残留；
- 生命周期跟 fd 引用计数绑定；
- 容器和 namespace 隔离更清楚；
- 可以通过 Unix domain socket 安全传递；
- 适合 RPC、sandbox、browser、多进程 runtime；
- 可配合 sealing 防止大小或内容被篡改。

典型结构： 

```text
producer
  memfd_create
  write / mmap fill data
  send fd via UDS

consumer
  recv fd
  mmap read data
  close when done
```

### 6.3 sealing

`memfd` 支持 seal 机制，限制后续修改：

```text
F_SEAL_SHRINK  防止缩小
F_SEAL_GROW    防止增大
F_SEAL_WRITE   防止写
F_SEAL_SEAL    防止再添加 seal
```

这适合传递只读大对象：

```text
create -> fill -> seal write/grow/shrink -> send fd -> read-only consumers
```

### 6.4 和 POSIX shm 的区别

| 维度 | POSIX shm | `memfd` |
| --- | --- | --- |
| 命名 | `/dev/shm/<name>` | fd，无路径 |
| 生命周期 | 需要 `shm_unlink` | fd 引用计数 |
| 传递方式 | 通过名字重新打开 | 通过 fd 传递 |
| 残留风险 | 有 | 低 |
| 容器隔离 | 依赖 `/dev/shm` mount | fd 语义更直接 |
| 适合场景 | 标准命名共享内存 | 现代 IPC / RPC / sandbox |

## 7. `hugetlbfs` 共享内存

`hugetlbfs` 也可以用作共享内存 backing，但它不属于普通 shmem/tmpfs。

```text
mount hugetlbfs
  -> create hugepage-backed file
  -> mmap MAP_SHARED in multiple processes
  -> shared huge pages
```

典型场景：

- 低延迟 IPC；
- 大页 ring buffer；
- DPDK hugepage memory；
- RDMA buffer；
- 数据库共享 buffer；
- 高性能队列。

优点：

- TLB 覆盖大；
- 映射稳定；
- 不受 THP collapse/split 影响；
- 适合长期 pinned 大块内存。

缺点：

- 需要预留 huge page pool；
- 运维复杂；
- 内存利用率风险高；
- 分配失败更硬；
- 不像 tmpfs 那样灵活回收。

## 8. 共享内存与大页组合

| 组合 | 底层 | 是否自动大页 | 是否可 swap | 适合场景 |
| --- | --- | --- | --- | --- |
| POSIX shm + 4K | `/dev/shm` tmpfs | 否或取决于策略 | 通常可 | 通用 IPC |
| tmpfs + THP | shmem/tmpfs | 可配置 | 通常可 | 大块共享缓存、AI 推理缓存 |
| memfd + THP | shmem-backed memfd | 可受 shmem THP 策略影响 | 通常可 | 现代 IPC、FD 传递 |
| hugetlbfs | huge page pool | 显式大页 | 否 | 低延迟、RDMA、DPDK、数据库 |
| System V shm + hugepage | `SHM_HUGETLB` 等 | 显式大页 | 否 | 老系统大页 IPC |

高性能服务常见决策树：

```text
需要路径和可观察文件？
  yes -> tmpfs / POSIX shm
  no  -> memfd

需要极低尾延迟和固定大页？
  yes -> hugetlbfs
  no  -> shmem/tmpfs/memfd + THP madvise

运行在容器中且需要 fd 传递？
  yes -> memfd
```

## 9. 生命周期和清理

### 9.1 System V

```text
shmget creates object
process attach/detach
object remains until IPC_RMID and no attachments
```

观测：

```bash
ipcs -m
ipcrm -m <shmid>
```

风险：进程退出后对象可能残留。

### 9.2 POSIX shm

```text
shm_open creates named object
shm_unlink removes name
mapping may live until unmapped
```

风险：忘记 `shm_unlink` 会在 `/dev/shm` 留对象。

### 9.3 tmpfs file

普通文件生命周期：

```text
open -> mmap -> unlink optional -> mappings keep pages alive
```

可以采用“创建后立即 unlink”的模式减少残留：

```text
open tmpfs file
unlink path
mmap still valid while fd/mapping alive
```

### 9.4 memfd

```text
memfd_create -> fd
fd passed around
last fd/mapping gone -> object released
```

无路径，残留风险低。

### 9.5 hugetlbfs

文件路径和 huge page pool 都要关注：

- 文件生命周期类似文件；
- huge page pool 是独立资源；
- 即使文件释放，预留 huge page 数量仍由系统配置决定。

## 10. 同步与一致性

共享内存不是消息队列。它只共享字节，不定义并发协议。

常见同步方式：

| 方式 | 特点 | 场景 |
| --- | --- | --- |
| process-shared pthread mutex | 标准、易用 | 低中频共享结构 |
| futex | 低层、高性能 | 自定义锁/队列 |
| eventfd | 通知机制 | producer/consumer 唤醒 |
| Unix domain socket | fd 传递 + 控制面 | memfd 传递 |
| lock-free ring | 极致性能 | 单生产者/单消费者或特定模型 |
| seqlock / RCU-like | 读多写少 | 配置、元数据 |

一个典型共享 ring buffer：

```text
shared memory layout

+----------------+----------------+-----------------------------+
| header         | ring metadata  | data slots                   |
| magic/version  | head/tail      | fixed-size or variable data  |
+----------------+----------------+-----------------------------+
        |                |
        |                +-- atomics / futex / eventfd
        +-- ABI compatibility
```

必须定义：

- 字节序和结构体对齐；
- version/magic；
- cache line padding；
- head/tail 的 memory ordering；
- 崩溃后的恢复策略；
- producer/consumer 生命周期。

## 11. 容器和 cgroup 视角

容器里共享内存最常见的问题是 `/dev/shm` 太小：

```text
docker default /dev/shm
  often 64 MiB
```

表现：

- `shm_open` 成功但写入时失败；
- mmap 后访问触发 `SIGBUS`；
- 多进程框架启动失败；
- 浏览器、数据库、AI runtime 共享内存不足。

检查：

```bash
df -h /dev/shm
mount | grep shm
```

修复：

```bash
docker run --shm-size=1g ...
```

或使用自定义 tmpfs mount / memfd / host IPC 策略，但要评估隔离风险。

## 12. 观测与调试

### 12.1 查看映射

```bash
cat /proc/<pid>/maps
cat /proc/<pid>/smaps
```

关注：

```text
/dev/shm/...
/memfd:...
/mnt/huge/...
Shared_Clean / Shared_Dirty
AnonHugePages
ShmemPmdMapped
KernelPageSize
MMUPageSize
VmFlags
```

### 12.2 查看 tmpfs 容量

```bash
df -h /dev/shm
df -h /run
mount | grep tmpfs
```

### 12.3 查看 System V IPC

```bash
ipcs -m
ipcrm -m <shmid>
```

### 12.4 查看 memfd

```bash
ls -l /proc/<pid>/fd
```

可能看到：

```text
/memfd:name (deleted)
```

### 12.5 查看 huge page

```bash
cat /proc/meminfo | grep -i huge
```

## 13. 终极对比表

| 机制 | 挂载点 | 路径 | fd 传递 | THP | hugetlb | swap | 生命周期 | 推荐场景 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `System V shm` | 不需要 | 无普通路径 | 不以 fd 为主 | 取决于实现/配置 | 可用特殊 flag | 取决于类型 | IPC 对象 | 遗留系统 |
| `POSIX shm` | `/dev/shm` | 固定 | 可传 fd 但通常按名打开 | 受 shmem 策略影响 | 否 | 通常可 | name + mapping | 通用 IPC |
| tmpfs file | 自定义 | 有 | 可 | 受 shmem 策略影响 | 否 | 通常可 | 文件 | 可控共享内存 |
| `memfd` | 不需要 | 无 | 强 | 受 shmem 策略影响 | 可通过 flag/系统支持 | 通常可 | fd 引用 | 现代 IPC/RPC |
| `hugetlbfs` | 需要 | 有 | 可 | 不适用 | 是 | 否 | 文件 + pool | 低延迟大页 IPC |

## 14. 工程选型

### 14.1 通用跨进程共享

优先：`POSIX shm_open` 或 tmpfs 文件。

```text
simple, named, debuggable
```

### 14.2 需要自定义容量和策略

优先：自定义 tmpfs mount。

```text
mount -t tmpfs -o size=... tmpfs /path
```

适合：

- 大块缓存；
- 多进程共享数据页；
- 需要独立容量控制；
- 需要配合 shmem THP 策略。

### 14.3 需要无路径、强隔离、fd 传递

优先：`memfd_create`。

适合：

- RPC 大对象传递；
- sandbox；
- 容器内多进程；
- browser/runtime；
- 零拷贝控制面 + 数据面分离。

### 14.4 需要固定大页和极低尾延迟

优先：`hugetlbfs`。

适合：

- DPDK；
- RDMA buffer；
- 低延迟队列；
- 数据库 buffer pool；
- 长生命周期共享大块内存。

### 14.5 AI 推理 / KV Cache 相关场景

如果是在 CPU 侧或跨进程共享大块缓存，可考虑：

```text
memfd / tmpfs
  + THP madvise
  + explicit lifecycle
  + perf validation
```

如果涉及 GPU 显存 KV Cache，Linux 共享内存只是 CPU/IPC 侧基础，不等同于 GPU memory allocator。需要另看 CUDA IPC、NCCL、RDMA、pinned memory 和推理引擎的 block manager。

## 15. 实验小结：共享内存机制怎么验证

这一组实验目标是验证三件事：

```text
1. 共享内存是否真的映射到多个进程。
2. POSIX shm / tmpfs / memfd 在 /proc 中分别长什么样。
3. 共享内存是否使用 THP，需要从 smaps/meminfo 观察，而不是只看 API。
```

### 15.1 实验一：`memfd_create` + Unix socket 传 fd

这个实验展示现代 IPC 常见模型：控制面用 Unix domain socket 传 fd，数据面用 `memfd + mmap` 共享页。

保存为 `memfd_pass.c`：

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/socket.h>
#include <sys/syscall.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#ifndef MFD_CLOEXEC
#define MFD_CLOEXEC 0x0001U
#endif

static int xmemfd_create(const char *name, unsigned int flags) {
    return (int)syscall(SYS_memfd_create, name, flags);
}

static void send_fd(int sock, int fd) {
    char buf[CMSG_SPACE(sizeof(fd))] = {0};
    struct iovec io = {.iov_base = (void *)"F", .iov_len = 1};
    struct msghdr msg = {.msg_iov = &io, .msg_iovlen = 1,
                         .msg_control = buf, .msg_controllen = sizeof(buf)};
    struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_RIGHTS;
    cmsg->cmsg_len = CMSG_LEN(sizeof(fd));
    memcpy(CMSG_DATA(cmsg), &fd, sizeof(fd));
    if (sendmsg(sock, &msg, 0) < 0) {
        perror("sendmsg");
        exit(1);
    }
}

static int recv_fd(int sock) {
    char c;
    char buf[CMSG_SPACE(sizeof(int))] = {0};
    struct iovec io = {.iov_base = &c, .iov_len = 1};
    struct msghdr msg = {.msg_iov = &io, .msg_iovlen = 1,
                         .msg_control = buf, .msg_controllen = sizeof(buf)};
    if (recvmsg(sock, &msg, 0) < 0) {
        perror("recvmsg");
        exit(1);
    }
    struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
    int fd;
    memcpy(&fd, CMSG_DATA(cmsg), sizeof(fd));
    return fd;
}

int main(void) {
    int sv[2];
    if (socketpair(AF_UNIX, SOCK_DGRAM, 0, sv) != 0) {
        perror("socketpair");
        return 1;
    }

    size_t len = 2 * 1024 * 1024;
    int fd = xmemfd_create("demo-shared-buffer", MFD_CLOEXEC);
    if (fd < 0) {
        perror("memfd_create");
        return 1;
    }
    if (ftruncate(fd, (off_t)len) != 0) {
        perror("ftruncate");
        return 1;
    }

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        close(sv[0]);
        int rfd = recv_fd(sv[1]);
        char *p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, rfd, 0);
        if (p == MAP_FAILED) {
            perror("child mmap");
            return 1;
        }
        printf("child pid=%ld sees: %s\n", (long)getpid(), p);
        strcpy(p + 4096, "reply from child");
        printf("child sleeping 20s, inspect /proc/%ld/maps\n", (long)getpid());
        fflush(stdout);
        sleep(20);
        return 0;
    }

    close(sv[1]);
    char *p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) {
        perror("parent mmap");
        return 1;
    }
    strcpy(p, "hello from parent");
    send_fd(sv[0], fd);
    sleep(1);
    printf("parent pid=%ld sees: %s\n", (long)getpid(), p + 4096);
    printf("parent sleeping 20s, inspect /proc/%ld/fd and smaps\n", (long)getpid());
    fflush(stdout);
    sleep(20);
    waitpid(pid, NULL, 0);
    return 0;
}
```

编译运行：

```bash
gcc -O2 -Wall -o memfd_pass memfd_pass.c
./memfd_pass
```

观察 fd：

```bash
ls -l /proc/<parent-pid>/fd | grep memfd
ls -l /proc/<child-pid>/fd | grep memfd
```

预期能看到类似：

```text
/memfd:demo-shared-buffer (deleted)
```

观察映射：

```bash
grep -n 'memfd:demo-shared-buffer' /proc/<pid>/maps
grep -A20 -B2 'memfd:demo-shared-buffer' /proc/<pid>/smaps
```

这个实验说明：

```text
1. memfd 没有路径，但有 fd 和 /proc 可见名字。
2. fd 可以通过 Unix socket 传给另一个进程。
3. 两个进程 mmap 后看到同一份底层页。
```

### 15.2 实验二：POSIX shm 与 `/dev/shm`

保存为 `posix_shm_demo.c`：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(void) {
    const char *name = "/demo-posix-shm";
    size_t len = 2 * 1024 * 1024;
    int fd = shm_open(name, O_CREAT | O_RDWR, 0600);
    if (fd < 0) {
        perror("shm_open");
        return 1;
    }
    if (ftruncate(fd, (off_t)len) != 0) {
        perror("ftruncate");
        return 1;
    }
    char *p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    strcpy(p, "hello posix shm");
    printf("pid=%ld mapped %s, inspect /dev/shm and /proc/%ld/maps\n",
           (long)getpid(), name, (long)getpid());
    fflush(stdout);
    sleep(30);
    munmap(p, len);
    close(fd);
    shm_unlink(name);
    return 0;
}
```

编译运行：

```bash
gcc -O2 -Wall -o posix_shm_demo posix_shm_demo.c -lrt
./posix_shm_demo
```

观察：

```bash
ls -lh /dev/shm/demo-posix-shm
grep demo-posix-shm /proc/<pid>/maps
grep -A20 -B2 demo-posix-shm /proc/<pid>/smaps
```

这个实验说明：

```text
POSIX shm object
  -> visible under /dev/shm
  -> tmpfs/shmem backing
  -> mmap MAP_SHARED into process
```

### 15.3 实验三：tmpfs 容量与 `SIGBUS` 风险

`tmpfs` 文件 `ftruncate` 到一个大小，不代表后续写入一定成功。真正消耗内存通常发生在写入或缺页时。

准备一个较小 tmpfs：

```bash
mkdir -p /tmp/smalltmpfs
sudo mount -t tmpfs -o size=64M tmpfs /tmp/smalltmpfs
```

写一个超过容量的文件并 mmap 触摸：

```bash
truncate -s 256M /tmp/smalltmpfs/bigfile
```

用 Python 快速触摸：

```bash
python3 - <<'PY'
import mmap
import os

path = "/tmp/smalltmpfs/bigfile"
fd = os.open(path, os.O_RDWR)
st = os.fstat(fd)
m = mmap.mmap(fd, st.st_size, mmap.MAP_SHARED, mmap.PROT_WRITE)
for off in range(0, st.st_size, 4096):
    m[off:off+1] = b"x"
print("done")
PY
```

可能结果：

```text
Bus error
```

实验意义：

```text
1. tmpfs size 是实际可用容量约束。
2. ftruncate 只是改变文件大小，不等于物理页已经准备好。
3. mmap 后写入时才可能暴露 ENOSPC/SIGBUS 类问题。
```

清理：

```bash
sudo umount /tmp/smalltmpfs
rmdir /tmp/smalltmpfs
```

如果没有 sudo 权限，可以只用 `/dev/shm` 做容量观察，不强行 mount。

### 15.4 实验四：观察 shmem THP

如果内核支持 shmem THP，可以用 tmpfs 文件加 `madvise(MADV_HUGEPAGE)` 观察 `ShmemPmdMapped`。

实验思路：

```text
tmpfs file
  -> ftruncate large size
  -> mmap MAP_SHARED
  -> madvise(MADV_HUGEPAGE)
  -> sequential touch
  -> inspect smaps
```

观测命令：

```bash
grep -A30 -B2 '/dev/shm/your-file' /proc/<pid>/smaps | \
  grep -E 'Size|KernelPageSize|MMUPageSize|ShmemPmdMapped|VmFlags'
```

预期：如果 shmem THP 生效，`ShmemPmdMapped` 可能大于 0。若为 0，不代表程序错误，可能是：

- 内核未开启 shmem THP；
- tmpfs mount 策略不允许；
- sysfs 策略不允许；
- 文件大小或对齐不合适；
- 内存碎片导致 fallback。

### 15.5 实验解读

这几组实验的判断重点：

| 实验 | 观察点 | 结论 |
| --- | --- | --- |
| `memfd` fd 传递 | `/proc/<pid>/fd`、`maps` | 无路径共享对象也能跨进程 mmap |
| POSIX shm | `/dev/shm`、`maps` | POSIX shm 是命名 tmpfs/shmem 对象 |
| tmpfs 容量 | `df -h`、`SIGBUS` | mmap 大小不等于可用物理页已保障 |
| shmem THP | `ShmemPmdMapped` | 共享内存也可能使用透明大页，但受策略影响 |

生产环境做共享内存选型时，至少要验证：

```text
1. 对象生命周期是否符合预期。
2. 容量限制是否符合业务峰值。
3. smaps 中映射来源是否正确。
4. THP/hugetlb 是否真的生效。
5. 同步协议是否能处理崩溃、短写、版本变化和并发竞争。
```

## 16. 常见误区

### 16.1 共享内存等于零拷贝？

共享内存减少进程间数据复制，但不代表整个链路没有拷贝。缺页、磁盘读入、设备 DMA、网络发送、GPU 拷贝仍可能发生。

### 16.2 `/dev/shm` 是无限内存？

不是。它是 tmpfs，有 mount size、cgroup 和系统内存限制。

### 16.3 `memfd` 不占内存？

不是。`memfd` 没有路径，但它的页仍占内存，必要时还可能受 swap 和 cgroup 限制。

### 16.4 hugetlbfs 和 tmpfs + THP 一样？

不一样。`hugetlbfs` 使用显式 huge page pool；tmpfs + THP 是 shmem 页在策略允许下使用透明大页。前者更稳定但运维重，后者更灵活但可能抖动。

### 16.5 mmap 后就已经分配物理内存？

通常不是。`mmap` 先建立 VMA，物理页多在首次访问时通过 page fault 分配或加载。是否预分配取决于 flags、文件系统、hugetlb、`MAP_POPULATE` 等因素。

## 17. 和页机制笔记的关系

本文回答“共享内存对象如何创建、命名、映射、释放”。前一篇 `03_Things_about_linux_pages_hugepages.md` 回答“这些映射背后的页大小、TLB、缺页和大页策略如何影响性能”。两者组合起来，才能解释现代高性能服务中的常见形态：

```text
shared memory API
  POSIX shm / tmpfs / memfd / hugetlbfs
        |
        v
page strategy
  4K / THP / mTHP / hugetlb
        |
        v
performance behavior
  copy avoidance / TLB coverage / latency / memory waste / isolation
```

## 参考资料

- Linux man-pages: `shmget(2)`  
  https://man7.org/linux/man-pages/man2/shmget.2.html
- Linux man-pages: `shmat(2)`  
  https://man7.org/linux/man-pages/man2/shmat.2.html
- Linux man-pages: `shm_open(3)`  
  https://man7.org/linux/man-pages/man3/shm_open.3.html
- Linux man-pages: `memfd_create(2)`  
  https://man7.org/linux/man-pages/man2/memfd_create.2.html
- Linux man-pages: `tmpfs(5)`  
  https://man7.org/linux/man-pages/man5/tmpfs.5.html
- Linux kernel documentation: tmpfs  
  https://docs.kernel.org/filesystems/tmpfs.html
- Linux kernel documentation: HugeTLB Pages  
  https://docs.kernel.org/admin-guide/mm/hugetlbpage.html

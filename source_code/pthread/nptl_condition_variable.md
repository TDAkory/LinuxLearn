# glibc NPTL condition variable：为什么需要 G1/G2 两组 waiter

> 上游仓库：[glibc.git](https://sourceware.org/git/glibc.git)  
> 分析基线：`glibc-2.44`，commit `c3a3a9808ad3ab4a3336836833f83288b672ccbf`  
> 验证日期：2026-10-14  
> 分析范围：Linux NPTL 的 `pthread_cond_wait/signal/broadcast`、timeout/cancellation 与销毁协调  
> 不展开：PI-aware requeue、其他 pthread 实现和历史版本算法

**一句话总结**：glibc condition variable 用 64-bit waiter sequence 决定哪些 waiter 有资格消费某次 signal，再用两个轮换的 32-bit futex word 承载当前可被 signal 的 G1 和仍在收集新 waiter 的 G2，从而绕开 futex 不保证 FIFO、单个 32-bit word 容易 ABA 的限制。

## 1. condition variable 等待的是 predicate，不是 signal

condition variable 本身不保存业务条件。正确用法始终是在关联 mutex 保护下检查 predicate：

```c
pthread_mutex_lock(&mutex);
while (!predicate())
    pthread_cond_wait(&cond, &mutex);
consume_state();
pthread_mutex_unlock(&mutex);
```

`pthread_cond_wait()` 必须提供三个语义阶段：

1. 释放 mutex 并进入等待；
2. 被 signal、broadcast、timeout、cancellation 或虚假事件解除等待；
3. 返回调用方前重新获得 mutex。

glibc 源码说明，这三个阶段以及 signal/broadcast 可以放入一个与程序 happens-before 一致的总序，但这个总序本身不额外创造 happens-before。这与 POSIX 允许虚假唤醒相容：wait 返回只表示“应该重新检查 predicate”，不证明某个特定 signal 与这个 waiter 一一对应。

## 2. 为什么不能只用一个 sequence futex

直觉实现可能是：

```text
waiter 读取 sequence → 释放 mutex → futex_wait(sequence, old)
signaler sequence++ → futex_wake(sequence, 1)
```

但 glibc 还必须解决“signal 只应该提供给 signal 发生前已经登记的 waiter”这一资格问题：

- futex 不保证按 waiter 登记顺序唤醒；
- 一个后来的 waiter 可能先被内核选中，抢走原本属于早期 waiter 的通知；
- futex word 是 32 bit，而长期运行的 waiter sequence 需要避免 wrap-around ABA；
- waiter 取得排号与真正进入 futex wait 不是一个原子动作；
- process-shared condvar 不能依赖仅在单进程堆中动态分配 waiter 队列。

glibc 因此把“资格顺序”和“内核阻塞位置”分开：

```text
64-bit __wseq
  └── 给每个 waiter 一个单调位置，定义 signal 的资格边界

两个 group slot
  ├── G1：已经有资格消费 signal 的有限 waiter 集合
  └── G2：G1 尚未结束期间新到达的 waiter，暂不接收 signal

两个 32-bit __g_signals[]
  └── 分别作为两个 group slot 的 futex word
```

两个 slot 会反复交换 G1/G2 角色，而不是为每一代 waiter 分配新内存。

## 3. `pthread_cond_t` 的核心状态

公共布局定义在 [`sysdeps/nptl/bits/thread-shared-types.h`](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/nptl/bits/thread-shared-types.h;hb=glibc-2.44)：

```c
struct __pthread_cond_s {
    __atomic_wide_counter __wseq;
    __atomic_wide_counter __g1_start;
    unsigned int __g_size[2];
    unsigned int __g1_orig_size;
    unsigned int __wrefs;
    unsigned int __g_signals[2];
    // 保留字段省略
};
```

| 字段 | 含义 | 并发访问方式 |
| --- | --- | --- |
| `__wseq` | waiter sequence；最低位记录当前 G2 使用哪个 slot，其余位是下一个 waiter 的位置 | waiter fetch-add 2；signaler 切组时 xor 最低位 |
| `__g1_start` | 当前 G1 起始位置；小于它的 waiter 属于已经关闭的旧组 | signaler 推进，waiter 观察 |
| `__g_size[2]` | 每组尚未 signal 的有效 waiter 数；G2 取消通过 unsigned 环绕表示负调整 | condvar 内部锁保护 |
| `__g1_orig_size` | G1 初始大小；最低两位复用为 condvar 内部 0/1/2 锁 | signal/broadcast/cancel 串行化 |
| `__g_signals[2]` | 每组仍可消费的 signal 位置，也是 waiter 真正 futex wait 的 word | waiter consume；signaler add + wake |
| `__wrefs` | waiter 引用计数及 shared/clock/destroy flags | wait 生命周期和 destroy 协调 |

`PTHREAD_COND_INITIALIZER` 把全部字段置零：G2 从位置 0 开始，初始 G1 为空且关闭。

### 3.1 `__wrefs` 的位布局

waiter 引用计数以 8 为单位增减，最低三位保存 flag：

```text
bit 0   process-shared
bit 1   clock：0 = CLOCK_REALTIME，1 = CLOCK_MONOTONIC
bit 2   destroy wake request
bit 3+  waiter reference count
```

这个计数不是“还没收到 signal 的 waiter 数”，而是仍可能访问 condvar 内存的 waiter 引用数。它使 `pthread_cond_destroy()` 可以等待所有已唤醒 waiter 确认不再访问对象。

## 4. wait：先取得位置并登记，再释放 mutex

核心路径在 [`nptl/pthread_cond_wait.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_wait.c;hb=glibc-2.44)：

```text
__pthread_cond_wait_common(cond, mutex)
  ├── wseq = fetch_add_acquire(__wseq, 2)
  │     g   = wseq & 1        // 登记时的 G2 slot
  │     seq = wseq >> 1       // waiter 的逻辑位置
  ├── __wrefs += 8            // 持有 condvar 生命周期引用
  ├── unlock mutex            // 登记完成后才释放
  ├── 循环：
  │     ├── 组已经关闭？seq < g1_start → 可以离开
  │     ├── 当前组有 signal？CAS consume 一个 → 可以离开
  │     └── futex_wait(__g_signals[g], observed)
  ├── __wrefs -= 8            // 确认不再等待
  └── 重新取得 mutex 后返回
```

### 4.1 登记必须发生在释放 mutex 之前

如果先释放 mutex，再增加 `__wseq`，持有同一 mutex 的 signaler 可能在空窗期完成：

```text
waiter unlock mutex
                         signaler lock mutex
                         predicate = true
                         cond_signal()  // 看不到 waiter
                         unlock mutex
waiter register + sleep
```

glibc 先取得 sequence 位置并增加 waiter reference，然后才通过 `__pthread_mutex_unlock_usercnt(mutex, 0)` 释放 mutex。正常程序中的 signaler持有同一 mutex，就能通过 mutex 的 happens-before 观察到这个 waiter 已经登记。

这里的“原子释放 mutex 并等待”不是单条指令，而是一套外部 mutex 和内部登记顺序共同建立的协议。

### 4.2 waiter 永远先加入当时的 G2

`__wseq` 的最低位给出当前 G2 slot。waiter 用 fetch-add 2 取得位置，不改变这个 slot bit：

```text
wseq before fetch-add = (next_seq << 1) | g2_slot
waiter position       = next_seq
waiter group slot     = g2_slot
```

只有 signaler 在关闭旧 G1、把旧 G2 提升为新 G1 时，才用 fetch-xor 翻转这个最低位。这样切组对并发登记的 waiter 是原子的：切换前取得位置的属于旧 G2，切换后到达的进入另一个新 G2。

## 5. G1/G2 的生命周期

```text
时间向右 ──────────────────────────────────────────────>

waiter sequence:  ... |------ current G1 ------|------ current G2 ------...
                        g1_start                 g2_start             wseq

signal：只填充 G1
new waiter：只加入 G2
G1 全部 signal 后：old G2 → new G1，old G1 slot → fresh G2
```

### 5.1 G1

G1 是一个边界已经固定的有限区间。`__g_size[g1]` 记录尚未获得 signal 的有效 waiter 数。signal 总是优先放入当前 G1，直到 size 归零。

### 5.2 G2

G2 的右边界随新 waiter 到达不断延伸，因此 signaler在切组前无法把它当成一个固定集合。取消或 timeout 的 G2 waiter 会对 `__g_size` 做负向调整；等 G2 被提升为 G1 时，再用 `wseq - new_g1_start` 计算原始区间大小并加上取消调整。

### 5.3 切换组角色

`__condvar_switch_g1()` 在 condvar 内部锁下执行：

1. 根据 `wseq` 和取消调整判断 G2 是否真的有 waiter；
2. 推进 `__g1_start`，关闭旧 G1；
3. release fetch-xor `__wseq` 的最低位，发布 G2 slot 切换；
4. old G2 成为 new G1；
5. 把该 slot 的 `__g_signals` 初始化到新的 `g1_start`；
6. 固定 new G1 的原始 size 和剩余 size。

等待线程用 acquire fetch-add 取得 `__wseq` 位置，与切组的 release xor 同步，避免看到半初始化的新 group。

## 6. signal：划定资格上界，只给 G1 增加一个 permit

[`nptl/pthread_cond_signal.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_signal.c;hb=glibc-2.44) 的路径是：

```text
__wrefs 中没有 waiter → 直接返回
  ↓
获取 condvar 内部锁
  ↓
读取 __wseq，得到 signal 观察到的 waiter 上界和当前 G1 slot
  ↓
若当前 G1 非空：
  __g_signals[g1]++
  __g_size[g1]--
否则若 G2 非空：
  切换 G1/G2
  给新 G1 增加一个 signal
  ↓
释放内部锁
  ↓
FUTEX_WAKE(__g_signals[g1], 1)
```

signal 对 `__wseq` 使用 relaxed load。源码给出的理由是：

- 若程序需要 signal 观察到某个 waiter，应通过关联 mutex 建立外部 happens-before；
- waiter 必须容忍虚假唤醒，不能从“我醒了”反推出某个特定 signal 与自己建立了 happens-before；
- signal 只需选择一个符合现有 happens-before 约束的序列位置。

signal 增加的是用户态 `__g_signals` permit；随后 futex wake 只是让可能睡在这个 group word 上的线程重新运行。被唤醒线程仍需通过 CAS 真正消费 signal，内核选择哪个 task 不决定最终资格。

## 7. waiter 如何判断可以离开

wait loop 每次观察两个量：

```c
signals = load_acquire(__g_signals[g]);
g1_start = load_relaxed(__g1_start);
```

有两条离开路径：

### 7.1 `seq < g1_start`

当前 G1 起点已经越过自己的位置，说明自己的旧 group 已经关闭，之前已有足够 signal 覆盖它。即使这个 waiter 太晚、尚未来得及在该 slot 上 futex wait，也不能再睡。

### 7.2 group 中存在可消费 signal

`__g_signals[g]` 保存相对当前 `g1_start` 的可用 signal 位置。若：

```c
(int)(signals - (unsigned int)g1_start) > 0
```

waiter 用 acquire CAS 把 signals 减一，成功才算消费 permit。CAS 失败说明其他 waiter 抢先消费，重新循环即可。

如果两条都不满足，waiter 才调用：

```c
futex_wait(__g_signals + g, observed_signals);
```

signal 在它入睡前改变了 word 时，内核 compare-and-block 会返回 `EAGAIN`；signal 在它入队后发生时，futex wake 使它重新检查上述用户态资格。

## 8. broadcast：完成旧 G1，再完成 old G2

[`nptl/pthread_cond_broadcast.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_broadcast.c;hb=glibc-2.44) 在一个内部临界区中完成三步：

1. 给旧 G1 剩余所有 waiter 增加 signal，并 wake-all 旧 G1；
2. 若 G2 非空，切换 group；
3. 给 old G2 / new G1 的所有 waiter 增加 signal，释放内部锁后 wake-all。

这等价于对 broadcast 发生前所有未解除等待的合格 waiter 发送足够多次 signal。broadcast 仍不意味着 predicate 为真，调用方仍然使用 while loop。

## 9. timeout 与 cancellation 为什么需要“补信号”

waiter timeout 或被取消时，不能简单从统计中消失，因为它可能与 signal 竞争。`__condvar_cancel_waiting()` 在内部锁下区分：

| waiter 所在阶段 | 处理方式 |
| --- | --- |
| 已关闭的旧 group | 视为已经消费过 signal，必须补发一个 replacement signal |
| 当前 G1，尚无专属于它的 signal | 减少 G1 remaining size |
| 当前 G1，size 已为 0 | 它可能拿走了最后一个 signal，补发 replacement signal |
| 当前 G2 | 减少 G2 有效 size；若计数将溢出，broadcast 制造一次干净状态 |

取消 cleanup 还会保守地额外 wake group futex。源码明确标为当前 cancellation 实现的补偿：线程可能在 syscall 返回后才收到取消，进而吞掉一个 futex wake；额外 wake 避免同组其他 waiter 永久睡眠。

完成取消登记后，cleanup 最后重新取得关联 mutex，保持 `pthread_cond_wait` 的取消语义。

## 10. 返回前为什么先减少 `__wrefs`，再锁 mutex

waiter 确认不再访问 group futex 后执行 `__condvar_confirm_wakeup()`，把 `__wrefs` 减 8，然后才 `__pthread_mutex_cond_lock(mutex)`。

这允许另一个线程在持有关联 mutex 时执行 `pthread_cond_destroy()`：destroy 设置 `__wrefs` 的 wake-request bit，并等待 waiter reference 归零。最后一个 waiter 用 release decrement 归零时 wake destroy。此时 waiter 仍可能在重新竞争外部 mutex，但已经不再访问 condvar 内存，所以 condvar 可以销毁。

```text
waiter 不再访问 condvar
  ↓
__wrefs -= 8，必要时 wake destroy
  ↓
重新取得外部 mutex
  ↓
pthread_cond_wait 返回
```

`__wrefs` 管的是 condvar 内存生命周期，不是 mutex ownership。

## 11. 与关联 mutex 的特殊协作

wait 释放 mutex 时使用：

```c
__pthread_mutex_unlock_usercnt(mutex, 0);
```

重新取得时使用：

```c
__pthread_mutex_cond_lock(mutex);
```

它们与公共 lock/unlock 的区别包括：

- 不增减 mutex 的 `__nusers`，使 mutex 生命周期检查仍能反映它与 active condvar waiter 的关联；
- cond-lock 路径使用 `lll_cond_lock()`，从取得锁开始就保留“可能有 futex waiter”的状态；
- robust mutex 会保留 `FUTEX_WAITERS`，避免在 condvar 相关交接中忘记潜在 waiter。

这说明 condition wait 不是“先调用一次普通 unlock，再调用 futex，最后普通 lock”这么简单；glibc 为 mutex 生命周期与潜在 waiter 状态提供了专门内部入口。

## 12. 这套算法保证什么，不保证什么

### 保证

- signal 不会被 signal 观察上界之后才登记的 waiter 合法消费；
- wait 的登记发生在释放关联 mutex 之前；
- 每个 group 的 signal permit 由用户态原子状态精确消费；
- timeout/cancellation 不会静默吞掉本应属于其他 waiter 的 signal；
- waiter 不再访问 condvar 后，destroy 可以安全复用其内存。

### 不保证

- 不保证唤醒顺序等同 waiter sequence 顺序；
- 不保证 signal 建立额外 happens-before；
- 不保证 wait 返回时 predicate 为真；
- 不保证 `pthread_cond_signal()` 一定导致某个特定 waiter 立刻获得 CPU 或 mutex；
- 不承诺没有虚假唤醒。

因此 predicate loop 和关联 mutex 仍是 API 正确性的必要部分，而不是对实现不足的临时补丁。

## 13. 源码阅读索引

| 问题 | 源码位置 |
| --- | --- |
| 算法总说明、wait、timeout、cancel | [`nptl/pthread_cond_wait.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_wait.c;hb=glibc-2.44) |
| signal 分配资格和 permit | [`nptl/pthread_cond_signal.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_signal.c;hb=glibc-2.44) |
| broadcast 三阶段 | [`nptl/pthread_cond_broadcast.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_broadcast.c;hb=glibc-2.44) |
| 内部锁和 G1/G2 切换 | [`nptl/pthread_cond_common.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_common.c;hb=glibc-2.44) |
| condvar ABI 布局 | [`sysdeps/nptl/bits/thread-shared-types.h`](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/nptl/bits/thread-shared-types.h;hb=glibc-2.44) |
| destroy 与 waiter reference | [`nptl/pthread_cond_destroy.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_cond_destroy.c;hb=glibc-2.44) |
| mutex 专用重锁入口 | [`nptl/pthread_mutex_cond_lock.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_mutex_cond_lock.c;hb=glibc-2.44) |

API 语义参考：

- [POSIX `pthread_cond_wait`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pthread_cond_clockwait.html)
- [POSIX `pthread_cond_signal` / `pthread_cond_broadcast`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pthread_cond_broadcast.html)

## 14. 结论

glibc condition variable 的复杂度主要来自四个事实：

1. signal 有资格边界，后来 waiter 不能抢走早先 signal；
2. futex 不保证 FIFO，不能直接用内核选择顺序表达资格；
3. futex word 只有 32 bit，长期 sequence 需要避免 ABA；
4. timeout、cancellation、process-shared 和 destroy 都要求 waiter 生命周期可追踪。

64-bit `__wseq` 负责顺序，G1/G2 负责把无限 waiter 流切成可 signal 的有限窗口，`__g_signals` 负责 permit 与 futex 阻塞，`__wrefs` 负责对象生命周期。关联 mutex 则提供业务 predicate 的 happens-before。它们共同实现了 POSIX 所说的“原子释放 mutex 并等待”，但任何一层都不能单独替代其他层。

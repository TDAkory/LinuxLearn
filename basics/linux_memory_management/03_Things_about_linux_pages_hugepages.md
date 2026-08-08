# Things about Linux 页机制与大页体系

> 本文整理 Linux 4K base page、页表、TLB、缺页、大页、`hugetlbfs`、`THP` 和 `mTHP` 的完整关系。重点不是背参数，而是建立一条工程判断链：页大小影响页表规模和 TLB 覆盖范围，页分配策略影响延迟抖动，业务应根据稳定性、内存利用率和运维成本选择 4K、静态大页或透明大页。

## 0. 一句话总览

Linux 内存页机制可以压成三层：

```text
用户态看到的虚拟地址
        |
        v
Linux 虚拟内存管理
  VMA / page table / page fault / reclaim / compaction
        |
        v
物理页框
  4K base page
  2M / 1G hugetlb page
  PMD-size THP
  mTHP multi-size anonymous page
```

大页的核心目标是扩大一次 TLB entry 覆盖的内存范围：

```text
4K page:
  1 TLB entry -> 4 KiB

2M huge page:
  1 TLB entry -> 2 MiB

1G huge page:
  1 TLB entry -> 1 GiB
```

收益来自更少的页表项、更少的 TLB miss、更少的缺页次数；风险来自连续物理内存要求、内存浪费、分配/清零/合并/拆分带来的延迟。

## 1. 4K base page：默认页机制

### 1.1 基础概念

Linux 以页为基本单位管理虚拟内存。常见 x86_64 环境下 base page 是 `4 KiB`。

```text
virtual address
  |
  | split into page number + offset
  v
page table walk
  |
  v
physical frame + offset
```

几个概念要分清：

| 概念 | 含义 |
| --- | --- |
| virtual page | 虚拟地址空间中的页 |
| physical frame | 物理内存中的页框 |
| page table | 虚拟页到物理页框的映射表 |
| PTE | page table entry，通常描述 4K 页映射 |
| PMD/PUD 等 | 多级页表中的中间层，也可承载大页映射 |
| TLB | CPU 缓存虚拟地址到物理地址翻译结果的结构 |
| page fault | 访问未建立物理映射或权限不满足时触发的异常 |

典型访问链路：

```text
CPU load/store
    |
    v
TLB lookup
    |
    +-- hit  -> physical address -> memory/cache
    |
    +-- miss -> page table walk
                   |
                   +-- valid mapping -> fill TLB -> access
                   |
                   +-- not present   -> page fault -> kernel handles
```

### 1.2 多级页表为什么会有开销

假设一个进程使用 `1 GiB` 连续匿名内存：

```text
1 GiB / 4 KiB = 262144 pages
```

这意味着需要大量 PTE 描述映射。页表本身也占内存，还会增加 page table walk 成本。

用 `2 MiB` 大页映射同样的 `1 GiB`：

```text
1 GiB / 2 MiB = 512 huge pages
```

映射项数量大幅下降。TLB 中每个 entry 覆盖的地址范围也更大。

### 1.3 为什么默认仍然是 4K

4K 页不是性能最好，而是通用性最好：

- 小对象、稀疏访问、短生命周期内存不容易浪费；
- 内存回收和换出粒度细；
- 页面权限、COW、NUMA 迁移、缺页处理粒度更灵活；
- 物理内存碎片压力低。

4K 的问题在大内存长生命周期场景中放大：

- 页表占用显著增加；
- TLB 覆盖范围不足；
- page fault 次数多；
- 页表修改可能带来 TLB shootdown；
- 大块内存扫描、预热、回收开销变高。

## 2. 为什么大页有效

### 2.1 TLB 覆盖范围

假设一个 CPU 的某级 TLB 能缓存 `N` 个 translation entry：

```text
4K pages:
  coverage = N * 4 KiB

2M pages:
  coverage = N * 2 MiB
```

相同 TLB entry 数量下，大页覆盖范围扩大数百倍。对大数组、大缓存、模型权重、KV Cache、数据库 buffer pool 这类顺序或局部访问的大块内存，TLB miss 会明显减少。

### 2.2 页表规模

```text
4K mapping for 1 GiB:
  262144 PTEs

2M mapping for 1 GiB:
  512 PMD-size mappings

1G mapping for 1 GiB:
  1 PUD-size mapping
```

页表项少，页表页占用少，page table walk 的层级和 cache 压力也更低。

### 2.3 缺页次数

首次访问匿名内存时，内核需要建立物理页映射并清零页面：

```text
4K:
  many small faults

2M:
  fewer faults, each fault handles larger range
```

这既可能是收益，也可能是风险。大页减少 fault 次数，但单次 fault 可能需要分配和清零更大的连续物理页，造成延迟毛刺。

## 3. Linux 页模型总览

| 机制 | 页大小 | 分配方式 | 是否透明 | 典型用途 |
| --- | --- | --- | --- | --- |
| base page | 通常 4K | 普通伙伴系统 | 是 | 通用业务 |
| `hugetlbfs` | 2M / 1G 等 | 预留或显式分配 huge page pool | 否 | 数据库、低延迟服务、显式大页 IPC |
| PMD-size THP | 常见 2M | 缺页或 `khugepaged` 动态分配/合并 | 是 | 大内存服务、堆、匿名内存、部分 shmem |
| `mTHP` | 多尺寸，如 16K~1M 级别 | 透明分配 | 是 | 降低 2M THP 的延迟和浪费风险 |

三类机制的核心区别：

```text
4K base page:
  fine-grained, flexible, default

hugetlbfs:
  explicit, reserved, stable, operationally heavy

THP / mTHP:
  automatic, flexible, may introduce background or fault-time latency
```

## 4. `hugetlbfs`：静态大页

### 4.1 原理

`hugetlbfs` 使用内核 huge page pool。大页可以在启动时预留，也可以在运行时尝试增加。

```text
boot / sysctl reserve
        |
        v
huge page pool
        |
        v
hugetlbfs file or MAP_HUGETLB mmap
        |
        v
process virtual mapping
```

与普通内存不同，`hugetlb` 页有强隔离特性：

- 预留后不参与普通内存回收；
- 不能像普通 page cache 那样随意换出；
- 分配失败通常直接失败，不像 THP 那样自然 fallback 到 4K；
- 需要提前规划数量、NUMA 和权限。

### 4.2 使用方式

常见方式一：挂载 `hugetlbfs`：

```bash
mount -t hugetlbfs none /mnt/huge
```

在该目录下创建文件，再 `mmap`：

```text
hugetlbfs file
  -> mmap
  -> huge page backed VMA
```

常见方式二：匿名映射时使用 `MAP_HUGETLB`：

```c
void *p = mmap(NULL, len,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
               -1, 0);
```

实际生产还要处理：

- huge page size；
- NUMA node；
- `RLIMIT_MEMLOCK` / capabilities；
- cgroup 限制；
- 预留失败和 fallback 策略。

### 4.3 优点

- 映射粒度稳定；
- TLB 覆盖范围大；
- 没有 `khugepaged` 合并抖动；
- 没有 THP fault-time compaction 的不确定性；
- 适合对尾延迟敏感的长期大块内存。

### 4.4 缺点

- 预留内存可能长期闲置；
- 运维复杂；
- 连续物理页要求高；
- 内存利用率不如普通页和 THP；
- 业务需要显式适配或配置。

### 4.5 适用场景

| 场景 | 是否适合 `hugetlbfs` | 原因 |
| --- | --- | --- |
| 数据库 buffer pool | 适合 | 大块长期内存，追求稳定 |
| 低延迟交易系统 | 适合 | 需要减少延迟毛刺 |
| DPDK / RDMA buffer | 常见 | 需要 pinned / huge / DMA-friendly 内存 |
| 普通 Web 服务 | 不适合 | 内存形态分散，预留浪费 |
| 动态多租户服务 | 谨慎 | 预留和隔离成本高 |

## 5. THP：Transparent Huge Page

### 5.1 设计目标

`THP` 的目标是让业务不显式使用 `hugetlbfs`，仍能获得大页收益。

```text
application malloc / mmap
        |
        v
anonymous VMA
        |
        +-- fault-time allocate huge page if possible
        |
        +-- khugepaged collapse 4K pages into THP
        |
        +-- split THP under pressure or special operations
```

它牺牲一部分可预测性，换取使用便利和更好的内存利用率。

### 5.2 运行模式

常见全局开关在：

```text
/sys/kernel/mm/transparent_hugepage/enabled
```

典型值：

```text
always
madvise
never
```

| 模式 | 含义 | 生产建议 |
| --- | --- | --- |
| `always` | 尽量对所有可用区域使用 THP | 容易引入不可控延迟和内存浪费 |
| `madvise` | 仅对 `MADV_HUGEPAGE` 区域使用 THP | 更适合生产精确控制 |
| `never` | 禁用 THP | 适合对 THP 抖动极敏感或收益不明确的服务 |

生产上更常见的策略是：

```text
global THP = madvise
business large long-lived region -> madvise(MADV_HUGEPAGE)
latency-sensitive small region   -> madvise(MADV_NOHUGEPAGE)
```

### 5.3 匿名内存 THP

匿名内存包括：

- heap；
- anonymous `mmap`；
- 大块 allocator arena；
- cache buffer；
- 一些运行时管理的大数组。

典型链路：

```text
malloc large buffer
  -> underlying mmap/brk
  -> VMA
  -> page fault
  -> THP allocation if policy allows
```

适合 THP 的匿名内存通常具备：

- 大块；
- 长生命周期；
- 密集访问；
- 写入后反复读取；
- 不频繁拆分、mprotect、fork COW。

### 5.4 shmem THP

`shmem` / `tmpfs` 也可以支持 THP，常见于：

- `/dev/shm`；
- tmpfs 文件；
- POSIX shared memory；
- 某些容器共享内存；
- memfd 的 shmem backing。

这使得“共享内存 + THP”成为高性能 IPC 和大内存缓存的一种组合。需要注意，shmem THP 受内核版本、mount 选项和 sysfs 策略影响。

### 5.5 `khugepaged`

`khugepaged` 是后台线程，用于扫描可合并区域，把多个 4K 页 collapse 成大页。

```text
4K pages in one VMA
  [p0][p1][p2] ... [p511]
        |
        | khugepaged scan + check
        v
2M THP
  [---------------- huge page ----------------]
```

它需要检查：

- VMA 是否允许 THP；
- 4K 页是否 present；
- 是否有不适合合并的页；
- NUMA 和内存策略；
- 空页数量是否可接受；
- 是否能分配目标大页。

收益：后台合并减少未来 TLB 压力。  
风险：后台 CPU 开销、内存搬迁、合并时延和不可预测性。

### 5.6 defrag 策略

THP 需要连续物理内存。内存碎片多时，内核可能进行 compaction。

常见策略在：

```text
/sys/kernel/mm/transparent_hugepage/defrag
```

概念上可以理解为：

| 策略 | 倾向 | 风险 |
| --- | --- | --- |
| `always` | 为了 THP 更积极整理内存 | fault 路径可能阻塞 |
| `defer` / `defer+madvise` | 更多交给后台整理 | THP 命中率可能下降 |
| `madvise` | 仅标记区域更积极 | 需要业务配合 |
| `never` | 不为 THP 做同步整理 | fallback 4K 更多 |

不同内核版本具体选项略有差异，生产上应以当前机器 sysfs 为准。

### 5.7 THP 的收益和风险

收益：

- TLB miss 降低；
- 页表内存减少；
- page fault 次数减少；
- 大块扫描和随机访问更稳定；
- 对大内存服务和模型推理可能提高吞吐。

风险：

- 首次 fault 清零 2M 页可能带来毛刺；
- compaction 可能阻塞业务；
- `always` 对稀疏访问内存可能浪费；
- `fork`、COW、`mprotect`、内存回收可能触发 split；
- 内存碎片严重时 fallback 到 4K，收益不稳定。

## 6. `mTHP`：multi-size Transparent Huge Page

传统 THP 常指 PMD-size `2 MiB` 大页。`mTHP` 扩展了透明大页的尺寸选择，允许更小的中间尺寸，例如 `16K`、`32K`、`64K`、`128K` 等，具体取决于架构和内核支持。

它解决的是 `4K` 和 `2M` 之间的断层：

```text
4K base page
  fine-grained, low waste, more TLB pressure

2M THP
  high TLB coverage, possible latency/waste

mTHP
  middle ground: moderate coverage, lower fault latency and waste
```

适用判断：

- 内存区域大，但不是每次都密集访问完整 2M；
- 需要减少 TLB miss，但不能接受 2M fault-time 毛刺；
- 业务希望更细粒度地平衡性能和浪费。

`mTHP` 是较新的机制，工程使用前要检查：

```text
/sys/kernel/mm/transparent_hugepage/hugepages-*/
```

并确认发行版内核、架构和容器环境是否支持。

## 7. 横向对比

| 维度 | 4K base page | `hugetlbfs` | THP | `mTHP` |
| --- | --- | --- | --- | --- |
| 使用方式 | 默认 | 显式预留/挂载/mmap | 自动或 `madvise` | 自动或策略控制 |
| 页大小 | 4K | 2M/1G 等 | 常见 2M | 多尺寸 |
| TLB 覆盖 | 低 | 高 | 高 | 中高 |
| 内存利用率 | 高 | 低到中 | 中 | 中高 |
| 延迟稳定性 | 稳定但 TLB 压力高 | 最稳定 | 可能有毛刺 | 比 2M THP 更温和 |
| 运维成本 | 低 | 高 | 中 | 中 |
| fallback | 不需要 | 通常失败 | 可 fallback 4K | 可 fallback 更小页 |
| 适合场景 | 普通服务 | 数据库/低延迟/DPDK/RDMA | 大内存服务/缓存/推理 | 新内核上的折中方案 |

## 8. 线上选型

### 8.1 普通服务

默认 4K 即可。除非观测到 TLB miss 或 page table 开销是瓶颈，不应为了“听起来更快”主动引入大页复杂度。

### 8.2 数据库和低延迟服务

优先考虑显式大页：

```text
hugetlbfs / MAP_HUGETLB
  + 固定预留
  + NUMA 规划
  + 启动时分配
```

原因是这类服务更在意尾延迟和可预测性，而不是完全自动化。

### 8.3 AI 推理、KV Cache、大内存缓存服务

优先评估：

```text
global THP = madvise
large cache / KV arena -> MADV_HUGEPAGE
small control objects  -> default 4K or MADV_NOHUGEPAGE
```

判断指标：

- `dTLB-load-misses`；
- page fault；
- `AnonHugePages` / `ShmemPmdMapped`；
- `THP` split / collapse；
- `TPOT`、p99 latency、吞吐；
- RSS 与实际访问密度。

### 8.4 DPDK / RDMA / pinned buffer

如果需要设备 DMA、pin memory、长期稳定映射，显式大页仍常见。原因不是只有 TLB，还是为了减少 pin 页数量、简化 IOMMU 映射和提升数据面稳定性。

## 9. 观测与验证

### 9.1 进程级

```bash
cat /proc/<pid>/smaps
```

关注：

```text
AnonHugePages
ShmemPmdMapped
KernelPageSize
MMUPageSize
VmFlags
```

### 9.2 系统级

```bash
cat /proc/meminfo | grep -E 'Huge|AnonHuge|Shmem'
```

常见字段：

```text
AnonHugePages
ShmemHugePages
HugePages_Total
HugePages_Free
Hugepagesize
```

### 9.3 sysfs 策略

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

`mTHP` 相关配置需要查看：

```bash
ls /sys/kernel/mm/transparent_hugepage/
```

### 9.4 perf 观测

```bash
perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses ./workload
```

如果问题是 TLB，应该看到：

- 大页开启后 TLB miss 降低；
- page fault 下降；
- p99/p999 是否改善要单独验证，不能只看平均值。

## 10. 实验小结：如何验证 THP/TLB/page fault

这一组实验的目标不是做权威 benchmark，而是建立可复现的观测方法：

```text
同一段大块匿名内存
  baseline 4K / default
  MADV_HUGEPAGE
  MADV_NOHUGEPAGE
        |
        v
观察：
  /proc/<pid>/smaps
  /proc/meminfo
  perf stat TLB/page-fault events
  wall time / p99 latency
```

### 10.1 最小实验程序：匿名大块内存触摸

保存为 `thp_touch.c`：

```c
#define _GNU_SOURCE
#include <errno.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>

static double now_sec(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (double)ts.tv_sec + (double)ts.tv_nsec / 1e9;
}

static void touch_memory(uint8_t *p, size_t len, size_t stride) {
    volatile uint64_t sum = 0;
    for (size_t off = 0; off < len; off += stride) {
        p[off]++;
        sum += p[off];
    }
    if (sum == 0xdeadbeef) {
        printf("sum=%lu\n", (unsigned long)sum);
    }
}

int main(int argc, char **argv) {
    size_t mib = argc > 1 ? strtoull(argv[1], NULL, 10) : 1024;
    const char *mode = argc > 2 ? argv[2] : "default";
    size_t len = mib * 1024UL * 1024UL;

    uint8_t *p = mmap(NULL, len, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (p == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    if (strcmp(mode, "huge") == 0) {
        if (madvise(p, len, MADV_HUGEPAGE) != 0) {
            perror("madvise(MADV_HUGEPAGE)");
        }
    } else if (strcmp(mode, "nohuge") == 0) {
        if (madvise(p, len, MADV_NOHUGEPAGE) != 0) {
            perror("madvise(MADV_NOHUGEPAGE)");
        }
    }

    printf("pid=%ld addr=%p len=%zu mode=%s\n",
           (long)getpid(), p, len, mode);
    fflush(stdout);

    double t0 = now_sec();
    touch_memory(p, len, 4096);
    double t1 = now_sec();
    touch_memory(p, len, 4096);
    double t2 = now_sec();

    printf("first_touch=%.6f sec second_touch=%.6f sec\n", t1 - t0, t2 - t1);
    printf("sleeping 30s, inspect /proc/%ld/smaps now\n", (long)getpid());
    fflush(stdout);
    sleep(30);

    munmap(p, len);
    return 0;
}
```

编译：

```bash
gcc -O2 -Wall -o thp_touch thp_touch.c
```

运行三组：

```bash
./thp_touch 1024 default
./thp_touch 1024 huge
./thp_touch 1024 nohuge
```

### 10.2 观察是否拿到 THP

程序会打印 `pid`，在它 sleep 时检查：

```bash
grep -A30 -B2 'rw-p' /proc/<pid>/smaps | grep -E 'Size|KernelPageSize|MMUPageSize|AnonHugePages|VmFlags'
```

更直接的做法是搜索大块 VMA 附近的 `AnonHugePages`：

```bash
cat /proc/<pid>/smaps | grep -E 'Size:|AnonHugePages:|KernelPageSize:|MMUPageSize:|VmFlags:'
```

预期现象：

```text
MADV_HUGEPAGE 生效时：
  AnonHugePages 可能接近触摸过的大块内存
  VmFlags 里可能出现 hg

MADV_NOHUGEPAGE 生效时：
  AnonHugePages 通常为 0
  VmFlags 里可能出现 nh
```

注意：`MADV_HUGEPAGE` 是建议，不是保证。内存碎片、sysfs 策略、cgroup、内核版本都会影响结果。

### 10.3 perf 对照 TLB 和 page fault

运行：

```bash
perf stat -e page-faults,dTLB-loads,dTLB-load-misses,cycles,instructions \
  ./thp_touch 1024 huge

perf stat -e page-faults,dTLB-loads,dTLB-load-misses,cycles,instructions \
  ./thp_touch 1024 nohuge
```

有参考意义的观察点：

| 指标 | 预期变化 | 含义 |
| --- | --- | --- |
| `page-faults` | THP 可能更少 | 单次 fault 覆盖更大范围 |
| `dTLB-load-misses` | THP 通常更低 | TLB 覆盖范围扩大 |
| `cycles` | 不一定稳定下降 | 受清零、compaction、CPU 频率影响 |
| first touch time | THP 可能更高或更低 | 大页分配/清零 vs fault 次数减少 |
| second touch time | THP 更可能体现收益 | 已建立映射后主要看 TLB/cache |

### 10.4 观察系统 THP 状态

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
cat /proc/meminfo | grep -E 'AnonHugePages|ShmemHugePages|HugePages|Hugepagesize'
```

如果支持 `mTHP`，可以看：

```bash
ls /sys/kernel/mm/transparent_hugepage/
find /sys/kernel/mm/transparent_hugepage -maxdepth 2 -type f -name enabled -print -exec cat {} \;
```

### 10.5 实验解读

这组实验能说明三件事：

```text
1. madvise 只是策略提示，不是强制分配。
2. THP 是否生效要看 smaps / meminfo，不要只看程序是否调用成功。
3. 性能收益要用 TLB miss、page fault、延迟一起判断。
```

不要只用一次运行结果下结论。更可靠的方式是：

```bash
for mode in default huge nohuge; do
  for i in $(seq 1 5); do
    perf stat -e page-faults,dTLB-load-misses ./thp_touch 1024 "$mode"
  done
done
```

如果 `huge` 模式没有明显收益，常见原因是：

- 机器全局 THP 策略不允许；
- 内存碎片导致 fallback；
- workload 不是 TLB-bound；
- 数据集太小，TLB 压力不足；
- first-touch 清零成本掩盖了后续访问收益；
- NUMA、CPU 频率、调度噪声影响实验。

## 11. 常见误区

### 11.1 大页一定更快？

不一定。稀疏访问、小对象、短生命周期内存可能浪费更多，甚至因为 compaction 或 split 引入抖动。

### 11.2 THP 和 `hugetlbfs` 是一回事？

不是。`hugetlbfs` 是显式、预留、强隔离的大页池；THP 是内核自动管理的透明大页，可 fallback、可 split、可由后台合并。

### 11.3 `always` 是生产最佳实践？

通常不是。`always` 容易把不适合的大量区域也纳入 THP。更稳妥的是 `madvise`，由业务标记大块长期内存。

### 11.4 THP 只支持匿名内存？

不是。THP 早期主要围绕匿名内存，现代内核也支持部分 shmem/tmpfs 场景。具体行为受内核版本、mount 选项和 sysfs 策略影响。

### 11.5 只看是否分配到大页就够？

不够。要同时看：

- TLB miss 是否下降；
- p99/p999 是否改善；
- RSS 是否膨胀；
- compaction 是否带来毛刺；
- split/collapse 是否频繁。

## 12. 和共享内存的关系

页机制决定性能边界，共享内存决定分配形态。高性能 IPC、缓存服务和推理服务常见组合是：

```text
tmpfs / shmem / memfd
        +
THP / mTHP / hugetlbfs
        =
共享、可映射、低拷贝、较高 TLB 覆盖的内存区域
```

下一篇 `04_Things_about_linux_shared_memory.md` 会从 `System V`、`POSIX shm_open`、`tmpfs/shmem`、`memfd_create`、`hugetlbfs` 共享内存的角度继续展开。

## 参考资料

- [Linux kernel documentation: HugeTLB Pages](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html)
- [Linux kernel documentation: Transparent Hugepage Support](https://docs.kernel.org/admin-guide/mm/transhuge.html)
- [Linux man-pages: `mmap(2)`](https://man7.org/linux/man-pages/man2/mmap.2.html)
- [Linux man-pages: `madvise(2)`](https://man7.org/linux/man-pages/man2/madvise.2.html)
- [Linux man-pages: `proc(5)`](https://man7.org/linux/man-pages/man5/proc.5.html)

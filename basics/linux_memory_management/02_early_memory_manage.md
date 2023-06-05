# [初期内存管理](https://zhuanlan.zhihu.com/p/444511088)

下表展示了总物理内存128MB的环境，内存布局如下：

```shell
0x07ffffff
    reserved
0x07dfffff
    usable
0x000fffff
    reserved(Motherboard BIOS 64 KiB is typical size)
0x000f0000
    reserved(video RAM ROM)
0x0009ffff
    reserved(EBDA Extened BIOS Data Area)
0x0009fbff  
    usable
0x00000000
```

历史原因，低1M的空间被划分为多个保留块，用于显存或ROM的保留内存，这是为了兼容8086或更早的CPU。

当下就有一个现实的问题摆在我们面前，如果这时候需要使用内存，该怎样管理起来呢？下面一起进入Linux初期物理内存管理的相关内容吧！

## [MemBlock内存管理器](https://github.com/torvalds/linux/blob/master/include/linux/memblock.h)

```cpp
/**
 * enum memblock_flags - definition of memory region attributes
 * @MEMBLOCK_NONE: no special request
 * @MEMBLOCK_HOTPLUG: memory region indicated in the firmware-provided memory
 * map during early boot as hot(un)pluggable system RAM (e.g., memory range
 * that might get hotunplugged later). With "movable_node" set on the kernel
 * commandline, try keeping this memory region hotunpluggable. Does not apply
 * to memblocks added ("hotplugged") after early boot.
 * @MEMBLOCK_MIRROR: mirrored region
 * @MEMBLOCK_NOMAP: don't add to kernel direct mapping and treat as
 * reserved in the memory map; refer to memblock_mark_nomap() description
 * for further details
 * @MEMBLOCK_DRIVER_MANAGED: memory region that is always detected and added
 * via a driver, and never indicated in the firmware-provided memory map as
 * system RAM. This corresponds to IORESOURCE_SYSRAM_DRIVER_MANAGED in the
 * kernel resource tree.
 */
enum memblock_flags {
	MEMBLOCK_NONE		= 0x0,	/* No special request */
	MEMBLOCK_HOTPLUG	= 0x1,	/* hotpluggable region */
	MEMBLOCK_MIRROR		= 0x2,	/* mirrored region */
	MEMBLOCK_NOMAP		= 0x4,	/* don't add to kernel direct mapping */
	MEMBLOCK_DRIVER_MANAGED = 0x8,	/* always detected via a driver */
};

/**
 * struct memblock_region - represents a memory region
 * @base: base address of the region
 * @size: size of the region
 * @flags: memory region attributes
 * @nid: NUMA node id
 */
struct memblock_region {
	phys_addr_t base;   // 内存区域的起始地址，类型为u64或u32，表示64位/32位架构的支持最大地址长度
	phys_addr_t size;   // 内存区域的大小
	enum memblock_flags flags;  // 内存区域的类型表示，相同类型的相邻内存，条件合适时可以被合并
#ifdef CONFIG_NUMA
	int nid;
#endif
};

/**
 * struct memblock_type - collection of memory regions of certain type
 * @cnt: number of regions
 * @max: size of the allocated array
 * @total_size: size of all regions
 * @regions: array of regions
 * @name: the memory type symbolic name
 */
struct memblock_type { 
	unsigned long cnt;  // 记录的内存区域（memblock_region）的数量
	unsigned long max;  // 最多能使用的内存区域数，当预留的内存区域不足时，管理器会扩展
	phys_addr_t total_size; // 所有内存区域的内存之和
	struct memblock_region *regions;    // 内存区域数组，每一项代表usable或保留的内存区域
	char *name; // 内存管理器类型的名称，例如“memory”，“reserved”等
};

/**
 * struct memblock - memblock allocator metadata
 * @bottom_up: is bottom up direction?
 * @current_limit: physical address of the current allocation limit
 * @memory: usable memory regions
 * @reserved: reserved memory regions
 */
struct memblock {
	bool bottom_up;  /* is bottom up direction? */ // 用于判断记录的内存是否从底部往顶部增长
	phys_addr_t current_limit;  // 当前内存管理器管理的物理地址上限
	struct memblock_type memory;    // 操作系统可用内存，即E820探测物理布局时，flags为usable的内存区域
	struct memblock_type reserved;  // 在boot阶段保留的内存，包括E820探测物理布局时，flags为reserved的内存区域、boot阶段分配出去的内存区域
};
```

## [Memblcok初始化](https://github.com/torvalds/linux/blob/master/mm/memblock.c)

```cpp
static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_MEMORY_REGIONS] __initdata_memblock;
static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_RESERVED_REGIONS] __initdata_memblock;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
static struct memblock_region memblock_physmem_init_regions[INIT_PHYSMEM_REGIONS];
#endif

// MemBlock内存管理器全局变量
struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,	/* empty dummy entry */
	.memory.max		= INIT_MEMBLOCK_MEMORY_REGIONS,
	.memory.name		= "memory",

	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,	/* empty dummy entry */
	.reserved.max		= INIT_MEMBLOCK_RESERVED_REGIONS,   // 系统预留reserved类型内存区域最大值，为128，不足时，会扩充
	.reserved.name		= "reserved",

	.bottom_up		= false,    // 内存管理器记录的内存区域的物理地址默认是从小到大排列，故这里是false
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,  // MemBlock内存管理器能管理最大物理内存，在64位系统是(uint64_t)-1
};

#ifdef CONFIG_ARCH_DISCARD_MEMBLOCK
    #define __init_memblock __meminit
    #define __initdata_memblock __meminitdata
#else
    #define __init_memblock
    #define __initdata_memblock
#endif

#define __meminitdata    __section(".meminit.data")
#define __initdata_memblock __meminitdata
```

也即所有被__initdata_memblock修饰的数据都会存放到内核镜像的.meminit.data节中。
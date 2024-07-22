# 从源码学习Linux中的数据结构(二)

> 本文内容基于 Linux v5.4 源码

## xarray

> `/include/linux/xarray.h`  See Documentation/core-api/xarray.rst for how to use the XArray.

XArray是一种抽象数据类型，表现为一个非常大的指针数组。它类似于哈希表或常规的可变数组，但提供了一些独特的优势：

* 允许以缓存高效的方式顺序访问条目。
* 增长数组时无需复制数据或更改内存管理单元（MMU）映射。
* 比使用标记指针的双向链表更节省内存、可并行化和缓存友好。


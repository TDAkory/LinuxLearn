# Linux Kernel的内存模型

- [Linux-Kernel Memory Model](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0124r3.html)
- [LINUX KERNEL MEMORY BARRIERS](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
- [A formal kernel memory-ordering model (part 1)](https://lwn.net/Articles/718628/)
- [A formal kernel memory-ordering model (part 2)](https://lwn.net/Articles/720550/)
- [Linux-Kernel Memory Ordering: Help Arrives At Last!](http://www.rdrop.com/users/paulmck/scalability/paper/LinuxMM.2017.01.19a.LCA.pdf)

## 从例子开始

在学习Linux源码的过程中，遇到了WRITE_ONCE宏，其在`linux v5.4`的定义如下。乍一看并不能理解为什么会这样实现，因此开始探究Linux内核的内存模型

```c
// /include/linux/compiler.h
#define WRITE_ONCE(x, val)                          \
({							                        \
	union { typeof(x) __val; char __c[1]; } __u =	\
		{ .__val = (__force typeof(x)) (val) };     \
	__write_once_size(&(x), __u.__c, sizeof(x));	\
	__u.__val;					                    \
})

static __always_inline void __write_once_size(volatile void *p, void *res, int size)
{
	switch (size) {
	case 1: *(volatile __u8 *)p = *(__u8 *)res; break;
	case 2: *(volatile __u16 *)p = *(__u16 *)res; break;
	case 4: *(volatile __u32 *)p = *(__u32 *)res; break;
	case 8: *(volatile __u64 *)p = *(__u64 *)res; break;
	default:
		barrier();
		__builtin_memcpy((void *)p, (const void *)res, size);
		barrier();
	}
}

// /include/linux/compiler-gcc.h
#define barrier() __asm__ __volatile__("": : :"memory")
// /include/linux/compiler-intel.h
#define barrier() __memory_barrier()
```

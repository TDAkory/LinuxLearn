# 从源码学习Linux中的数据结构

> 本文内容基于 Linux v5.4 源码

## 双向链表

核心文件：`include/linux/list.h`

### 基本结构

```c
// /include/linux/types.h
struct list_head {
	struct list_head *next, *prev;
};

// struct hlist_head {
// 	struct hlist_node *first;
// };

// struct hlist_node {
// 	struct hlist_node *next, **pprev;
// };
```

双链表的基础结构仅有两个指针，一个指向前一个节点，一个指向后一个节点。其保存数据的思路是将链表结构嵌入在实际的数据结构中，同时可以根据二者在内存分布中的位置关系，通过指针偏移来访问数据。

```c
// /include/linux/list.h
/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) container_of(ptr, type, member)

// /include/linux/kernel.h
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({				            \
	void *__mptr = (void *)(ptr);					                \
	BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&	\
			 !__same_type(*(ptr), void),			                \
			 "pointer type mismatch in container_of()");	        \
	((type *)(__mptr - offsetof(type, member))); })

// /include/linux/compiler_types.h
/* Are two types/vars the same type (ignoring qualifiers)? */
#define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))

// /include/linux/stddef.h
#ifdef __compiler_offsetof
#define offsetof(TYPE, MEMBER)	__compiler_offsetof(TYPE, MEMBER)
#else
#define offsetof(TYPE, MEMBER)	((size_t)&((TYPE *)0)->MEMBER)
#endif
```

访问的方法是通过 `list_entry` 宏来实现的，抛掉其内部的指针类型校验，**<font color=red>可以简单的将宏展开结果等价于 `(type *)((char *)(ptr) - (char *) &((type *)0)->member)`</font>**。这里的基本思路就是假设0地址是`type`类型的起始地址，访问`type->member`的地址就是`member`在`type`中的偏移量(因为基地址是0，所以这里其实隐含了减去基地址的逻辑)，即`list_head`在结构中的偏移，因此将真实的`list_head`地址减去偏移量，就能得到数据的真实起始地址。

## Ref

- [内核数据结构](https://mickyching.github.io/kernel/linux-kernel-data-structures.html)
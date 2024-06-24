# 从源码学习Linux中的数据结构

> 本文内容基于 Linux v5.4 源码

## 双向链表

> 核心文件：`include/linux/list.h`

```
/*
 * Simple doubly linked list implementation.
 *
 * Some of the internal functions ("__xxx") are useful when
 * manipulating whole lists rather than single entries, as
 * sometimes we already know the next/prev entries and we can
 * generate better code by using them directly rather than
 * using the generic single-entry routines.
 */
```

### 基本结构

```c
// /include/linux/types.h
struct list_head {
	struct list_head *next, *prev;
};
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

### 常用接口

**初始化**很简单的通过宏将头尾指针都指向了自己

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

// 和LIST_HEAD_INIT功能一样，这里传递的是指针，而前面传递的是对象
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	WRITE_ONCE(list->next, list);
	list->prev = list;
}
```

**添加**分为了添加在目标节点的尾部 list_add，或添加在目标节点的头部 list_add_tail，两者的实现都是 __list_add, 通过确认操作节点的prev next，完成三者指针关系的改写。

```c
/*
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	if (!__list_add_valid(new, prev, next))
		return;

	next->prev = new;
	new->next = next;
	new->prev = prev;
	WRITE_ONCE(prev->next, new);
}

static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}

static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
}
```

**删除**也很好理解，需要注意的是，`LIST_POISON1` 和 `LIST_POISON2` 是内部定义的两个非空指针。因为空指针可能导致程序崩溃，这里采用非空的预设指针来校验 entry 是否被正确的初始化。

```c
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	WRITE_ONCE(prev->next, next);
}

static inline void __list_del_entry(struct list_head *entry)
{
	if (!__list_del_entry_valid(entry))
		return;

	__list_del(entry->prev, entry->next);
}

static inline void list_del(struct list_head *entry)
{
	__list_del_entry(entry);
	entry->next = LIST_POISON1;
	entry->prev = LIST_POISON2;
}
```

**判空**

```c
static inline int list_empty(const struct list_head *head)
{
	return READ_ONCE(head->next) == head;
}
```

**遍历**，需要注意的是，在内核的使用中，链表头是不会嵌入到具体的数据结构中的，这主要是出于通用、灵活、性能等方面的考虑，因此获取链表的头尾元素，需要额外的操作，当然其本质还是对`list_entry`的调用。

```c
#define list_first_entry(ptr, type, member) \
	list_entry((ptr)->next, type, member)

#define list_last_entry(ptr, type, member) \
	list_entry((ptr)->prev, type, member)

#define list_for_each(pos, head) \
	for (pos = (head)->next; pos != (head); pos = pos->next)

#define list_for_each_prev(pos, head) \
	for (pos = (head)->prev; pos != (head); pos = pos->prev)
```

同时可以看到，这里的遍历其实就是for循环条件的等价替换，可以预见在循环的同时，进行节点的删除或修改，是不够安全的，可能导致循环的终止或访问到非预期的内存位置。

针对这种情况，内核实现也提供了安全的遍历接口，思路是提前保存当前节点的下一个节点，从而规避对当前节点修改时可能对尚未遍历的链表结构造成影响：

```c
/**
 * list_for_each_safe - iterate over a list safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop cursor.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_safe(pos, n, head) \
	for (pos = (head)->next, n = pos->next; pos != (head); \
		pos = n, n = pos->next)

#define list_for_each_prev_safe(pos, n, head) \
	for (pos = (head)->prev, n = pos->prev; \
	     pos != (head); \
	     pos = n, n = pos->prev)
```

当然遍历也包括定位当前数据节点的相邻节点，这里也需要指出 `list_head` 在数据结构中的位置。

```c
/**
 * list_next_entry - get the next element in list
 * @pos:	the type * to cursor
 * @member:	the name of the list_head within the struct.
 */
#define list_next_entry(pos, member) \
	list_entry((pos)->member.next, typeof(*(pos)), member)

#define list_prev_entry(pos, member) \
	list_entry((pos)->member.prev, typeof(*(pos)), member)
```

## 哈希链表

> `/include/linux/list.h`

### 基本结构

```c
// /include/linux/types.h
struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

`hlist_head` 和 `hlist_node` 是内核哈希链表的头节点和数据节点，其结构如下图所示。`hlist_head` 在哈希桶中，仅包含一个指向链表的指针，数据节点则都分布在链表上，包含前驱和后继两个指针。

特别的是，数据节点 `hlist_node` 是一个二级指针，指向的是前一个节点的 `next` 指针。之所以这么做，是因为链表首个元素的前驱是 `hlist_head`，这种情况下，链表中的`prev`指针无法做到统一表达，因此需要用二级指针来屏蔽这种差异。

![hash_table](https://raw.githubusercontent.com/TDAkory/ImageResources/main/img/hlist_node.png)

此外对哈希表节点的数据访问和双向链表的数据访问方式完全一致，通过 `hlist_entry` 宏即可实现：

```c
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)
```

### 常用接口

**初始化**，这里的注释也可以看到，hlist_head 设计为单指针主要是为了节约内存占用，缺陷就是无法通过一次访问找到链表尾结点。不过这个需求在哈希表中并不是很常用。

```c
/*
 * Double linked lists with a single pointer list head.
 * Mostly useful for hash tables where the two pointer list head is
 * too wasteful.
 * You lose the ability to access the tail in O(1).
 */

#define HLIST_HEAD_INIT { .first = NULL }
#define HLIST_HEAD(name) struct hlist_head name = {  .first = NULL }
#define INIT_HLIST_HEAD(ptr) ((ptr)->first = NULL)
static inline void INIT_HLIST_NODE(struct hlist_node *h)
{
	h->next = NULL;
	h->pprev = NULL;
}
```

**添加**提供了几种常用实现，添加到链表头、链表元素的前面、链表元素的后面。

```c
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
	struct hlist_node *first = h->first;
	n->next = first;
	if (first)
		first->pprev = &n->next;
	WRITE_ONCE(h->first, n);
	n->pprev = &h->first;
}

/* next must be != NULL */
static inline void hlist_add_before(struct hlist_node *n,
					struct hlist_node *next)
{
	n->pprev = next->pprev;
	n->next = next;
	next->pprev = &n->next;
	WRITE_ONCE(*(n->pprev), n);
}

static inline void hlist_add_behind(struct hlist_node *n,
				    struct hlist_node *prev)
{
	n->next = prev->next;
	prev->next = n;
	n->pprev = &prev->next;

	if (n->next)
		n->next->pprev  = &n->next;
}
```

**删除**，把当前节点从链表中移除

```c
static inline void __hlist_del(struct hlist_node *n)
{
	struct hlist_node *next = n->next;
	struct hlist_node **pprev = n->pprev;

	WRITE_ONCE(*pprev, next);
	if (next)
		next->pprev = pprev;
}

static inline void hlist_del(struct hlist_node *n)
{
	__hlist_del(n);
	n->next = LIST_POISON1;
	n->pprev = LIST_POISON2;
}
```

**判空**

```c
static inline int hlist_empty(const struct hlist_head *h)
{
	return !READ_ONCE(h->first);
}
```

**遍历**和双向链表工作原理一样，不过实现上要复杂一点，并且没有反向迭代的版本

```c
/**
 * hlist_for_each_entry	- iterate over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the hlist_node within the struct.
 */
#define hlist_for_each_entry(pos, head, member)				\
	for (pos = hlist_entry_safe((head)->first, typeof(*(pos)), member);\
	     pos;							\
	     pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))

/**
 * hlist_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @pos:	the type * to use as a loop cursor.
 * @n:		another &struct hlist_node to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the hlist_node within the struct.
 */
#define hlist_for_each_entry_safe(pos, n, head, member) 		\
	for (pos = hlist_entry_safe((head)->first, typeof(*pos), member);\
	     pos && ({ n = pos->member.next; 1; });			\
	     pos = hlist_entry_safe(n, typeof(*pos), member))
```

## 队列

> 核心文件 /include/linux/kfifo.h 

A generic kernel FIFO implementation

### 基本结构

```c
struct __kfifo {
	unsigned int	in;
	unsigned int	out;
	unsigned int	mask;
	unsigned int	esize;
	void		*data;
};
```

内核FIFO的实现非常高效，它使用一个环形数组来存储数据，并且使用两个指针来标记队列的入口和出口

* in标记入队索引，入队n个数据时，in变量就+n
* out标记出队索引，出队k个数据时，out变量就+k
* out不允许大于in（out等于in时表示fifo为空）
* in不允许比out大超过fifo空间
* in、out都会保持增长，不会轻易从零重新计算
* 如果需要获取位置，则in、out会按位与mask，mask的大小是 size - 1，同时会要求FIFO的大小必须是2的幂次，保证了获取位置的效率

```c
#define __STRUCT_KFIFO_COMMON(datatype, recsize, ptrtype) \
	union { \
		struct __kfifo	kfifo; \
		datatype	*type; \
		const datatype	*const_type; \
		char		(*rectype)[recsize]; \
		ptrtype		*ptr; \
		ptrtype const	*ptr_const; \
	}

#define __STRUCT_KFIFO(type, size, recsize, ptrtype) \
{ \
	__STRUCT_KFIFO_COMMON(type, recsize, ptrtype); \
	type		buf[((size < 2) || (size & (size - 1))) ? -1 : size]; \
}

#define STRUCT_KFIFO(type, size) \
	struct __STRUCT_KFIFO(type, size, 0, type)

/**
 * DECLARE_KFIFO - macro to declare a fifo object
 * @fifo: name of the declared fifo
 * @type: type of the fifo elements
 * @size: the number of elements in the fifo, this must be a power of 2
 */
#define DECLARE_KFIFO(fifo, type, size)	STRUCT_KFIFO(type, size) fifo

#define __STRUCT_KFIFO_PTR(type, recsize, ptrtype) \
{ \
	__STRUCT_KFIFO_COMMON(type, recsize, ptrtype); \
	type		buf[0]; \
}

#define STRUCT_KFIFO_PTR(type) \
	struct __STRUCT_KFIFO_PTR(type, 0, type)

/**
 * DECLARE_KFIFO_PTR - macro to declare a fifo pointer object
 * @fifo: name of the declared fifo
 * @type: type of the fifo elements
 */
#define DECLARE_KFIFO_PTR(fifo, type)	STRUCT_KFIFO_PTR(type) fifo
```

宏定义提供了两种方式来声明一个FIFO结构，分别是静态的 STRUCT_KFIFO 和 动态的 STRUCT_KFIFO_PTR。

以 STRUCT_KFIFO 为例，假设我们定义一个存储int型数据的FIFO，大小为16，则最终的展开结果是：

```c
// STRUCT_KFIFO(my_fifo, int, 16)
//      |
// STRUCT_KFIFO(int, 16) my_fifo
//      |
// struct __STRUCT_KFIFO(int, 16, 0, int) my_fifo
//      |
// struct {
// 	__STRUCT_KFIFO_COMMON(int, 0, int); 
// 	int buf[((16 < 2) || (16 & (16 - 1))) ? -1 : 16];
// } my_fifo

struct {
	union {
		struct __kfifo	kfifo;
		int		*type;
		const int	*const_type;
		int		(*rectype)[0];
		int		*ptr;
		int const	*ptr_const;
	};
	int buf[16];
} my_fifo;
```

上述展开结果中，存在通过 __STRUCT_KFIFO_COMMON 展开的一个 union，为什么这么设计呢？因为 __kfifo 中并不包含元素数据类型，元素指针类型，那如何确定这些类型呢？这些类型实质上是保存在了 union 的声明中，比如可以通过 typeof(*my_fifo->type) 来获取到 my_fifo 的数据类型是 int 类型。同时这些额外的声明（type、const_typ，etc）放在union内，也不会占用 my_fifo 的运行期开销。

动态的声明，唯一的区别就是不会立即指定buf大小，而是用一个不定长数组 buf[0] 来表示。

随之而来的问题是如果给出了一个 FIFO 的变量名，如何确定是通过哪一种方式声明的呢？内核也提供了判断方法，其判断是根据FIFO的数据是否和其结构存放在一起：

```c
/*
 * helper macro to distinguish between real in place fifo where the fifo
 * array is a part of the structure and the fifo type where the array is
 * outside of the fifo structure.
 */
#define	__is_kfifo_ptr(fifo) \
	(sizeof(*fifo) == sizeof(STRUCT_KFIFO_PTR(typeof(*(fifo)->type))))
```

### 初始化

```c
/**
 * INIT_KFIFO - Initialize a fifo declared by DECLARE_KFIFO
 * @fifo: name of the declared fifo datatype
 */
#define INIT_KFIFO(fifo) \
(void)({ \
	typeof(&(fifo)) __tmp = &(fifo); \
	struct __kfifo *__kfifo = &__tmp->kfifo; \
	__kfifo->in = 0; \
	__kfifo->out = 0; \
	__kfifo->mask = __is_kfifo_ptr(__tmp) ? 0 : ARRAY_SIZE(__tmp->buf) - 1;\
	__kfifo->esize = sizeof(*__tmp->buf); \
	__kfifo->data = __is_kfifo_ptr(__tmp) ?  NULL : __tmp->buf; \
})
```

## Ref

- [内核数据结构](https://mickyching.github.io/kernel/linux-kernel-data-structures.html)
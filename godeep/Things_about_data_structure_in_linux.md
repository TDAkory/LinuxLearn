# 从源码学习Linux中的数据结构

> 本文内容基于 Linux v5.4 源码

## 双向链表

核心文件：`include/linux/list.h`

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

```c
// /include/linux/types.h
struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

## Ref

- [内核数据结构](https://mickyching.github.io/kernel/linux-kernel-data-structures.html)
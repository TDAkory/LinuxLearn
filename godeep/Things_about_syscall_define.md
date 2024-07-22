# SYSCALL_DEFINE

- [SYSCALL\_DEFINE](#syscall_define)
  - [系统调用定义宏](#系统调用定义宏)
  - [系统调用的一般链路](#系统调用的一般链路)
  - [系统调用的展开](#系统调用的展开)
  - [系统调用表](#系统调用表)
  - [`SYSCALL_DEFINE0`](#syscall_define0)
  - [`SYSCALL_DEFINEn`](#syscall_definen)
  - [`__X64_SYS_STUBx`](#__x64_sys_stubx)
  - [Example of expansion of `write`](#example-of-expansion-of-write)

> include/linux/syscalls.h
> 
> linux-6.7
>
> [Linux Kernel - System Call](https://blog.jm233333.com/linux-kernel/system-call/#syscall-definition)
> 
> [Adding a New System Call](https://www.kernel.org/doc/html/latest/process/adding-syscalls.html?highlight=syscall_define)
>
> [linux v5.4 syscall](../../BlogSrc/how_to_define_a_syscall.md)

## 系统调用定义宏

```cpp
#ifndef SYSCALL_DEFINE0
#define SYSCALL_DEFINE0(sname)					\
    SYSCALL_METADATA(_##sname, 0);				\
    asmlinkage long sys_##sname(void);			\
    ALLOW_ERROR_INJECTION(sys_##sname, ERRNO);		\
    asmlinkage long sys_##sname(void)
#endif /* SYSCALL_DEFINE0 */

#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)				\
    SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
    __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

/*
 * The asmlinkage stub is aliased to a function named __se_sys_*() which
 * sign-extends 32-bit ints to longs whenever needed. The actual work is
 * done within __do_sys_*().
 */
#ifndef __SYSCALL_DEFINEx
#define __SYSCALL_DEFINEx(x, name, ...)					\
    __diag_push();							\
    __diag_ignore(GCC, 8, "-Wattribute-alias",			\
              "Type aliasing is used to sanitize syscall arguments");\
    asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
        __attribute__((alias(__stringify(__se_sys##name))));	\
    ALLOW_ERROR_INJECTION(sys##name, ERRNO);			\
    static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
    asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
    asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
    {								\
        long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
        __MAP(x,__SC_TEST,__VA_ARGS__);				\
        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
        return ret;						\
    }								\
    __diag_pop();							\
    static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
#endif /* __SYSCALL_DEFINEx */
```

对系统调用的定义，kernel是支持覆写的，在 include/linux/syscalls.h 头部有如下定义，允许覆盖 `SYSCALL_DEFINE0()` 和 `__SYSCALL_DEFINEx()`，同时调整了头文件路径(arch/x86/include/asm/syscall_wrapper.h)，由此可以达到定义覆写的目的。

> 宏的变更 https://lore.kernel.org/lkml/20180330093720.6780-3-linux@dominikbrodowski.net/


```c
#ifdef CONFIG_ARCH_HAS_SYSCALL_WRAPPER
/*
 * It may be useful for an architecture to override the definitions of the
 * SYSCALL_DEFINE0() and __SYSCALL_DEFINEx() macros, in particular to use a
 * different calling convention for syscalls. To allow for that, the prototypes
 * for the sys_*() functions below will *not* be included if
 * CONFIG_ARCH_HAS_SYSCALL_WRAPPER is enabled.
 */
#include <asm/syscall_wrapper.h>
#endif /* CONFIG_ARCH_HAS_SYSCALL_WRAPPER */
```


## 系统调用的一般链路

- 对于x86_64，当出现系统调用时，执行从用户态进入内核态并调用`entry_SYSCALL_64`
  - arch/x86/entry/entry_64.S
- 首先, `entry_SYSCALL_64` 调用 `do_syscall_64`, 由其继续调用 `do_syscall_x64`.
  
    ```unix
    call	do_syscall_64		/* returns with IRQs disabled */
    ```

  - `do_syscall_64` 和 `do_syscall_x64` 定义在 arch/x86/entry/common.c
  - `do_syscall_x64` 的核心就是查找调用表，传递参数执行调用，其执行结果会指标保存在寄存器中

    ```c
    static __always_inline bool do_syscall_x64(struct pt_regs *regs, int nr)
    {
        /*
         * Convert negative numbers to very high and thus out of range
         * numbers for comparisons.
         */
        unsigned int unr = nr;

        if (likely(unr < NR_syscalls)) {
            unr = array_index_nospec(unr, 	NR_syscalls);
            regs->ax = sys_call_table[unr]	(regs);
            return true;
        }
        return false;
    }
    ```

- Then, `do_syscall_x64` will call the corresponding function (e.g. __x64_sys_write) through the mapping array `sys_call_table` and the syscall id passed from the user space.
- Finally, the execution returns to user mode (see the end of entry_SYSCALL_64).

## 系统调用的展开

以 write 为例， 其函数定义在 fs/read_write.c

```cpp
ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count)
{
    ...
}

SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
{
    return ksys_write(fd, buf, count);
}
```

结合上面的宏定义，展开后的系统调用如下：

```cpp
static long __se_sys_write(long fd, long long buf, long count);
static inline long __do_sys_write(unsigned int fd, const char __user * buf, size_t count);
long __x64_sys_write(const struct pt_regs * regs);

long __x64_sys_write(const struct pt_regs * regs)
{
    return __se_sys_write(regs->di, regs->si, regs->dx);
}
static long __se_sys_write(long fd, long long buf, long count)
{
    long ret = __do_sys_write((__force unsigned int) fd, (__force const char __user *) buf, (__force size_t) count);
    return ret;
}
static inline long __do_sys_write(unsigned int fd, const char __user * buf, size_t count) {
    return ksys_write(fd, buf, count);
}
ssize_t ksys_write(unsigned int fd, const char __user * buf, size_t count) {
    // ...
}
```

如果查看系统调用的调用栈，可以看到 

```shell
(gdb) bt
#0  ksys_write (fd=1, buf=0x862ce0 "\nhello, busybox!\n\n", count=18) at fs/read_write.c:638
#1  __do_sys_write (count=18, buf=0x862ce0 "\nhello, busybox!\n\n", fd=1) at fs/read_write.c:659
#2  __se_sys_write (count=18, buf=8793312, fd=1) at fs/read_write.c:656
#3  __x64_sys_write (regs=0xffffc9000000bf58) at fs/read_write.c:656
#4  0xffffffff811383eb in do_syscall_x64 (nr=<optimized out>, regs=0xffffc9000000bf58) at arch/x86/entry/common.c:50
#5  do_syscall_64 (regs=0xffffc9000000bf58, nr=<optimized out>) at arch/x86/entry/common.c:80
#6  0xffffffff81200065 in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:118
#7  0x0000000000000000 in ?? ()
```

## 系统调用表

系统调用表定义在 arch/x86/entry/syscall_64.c

```c
#include <asm/syscall.h>

#define __SYSCALL(nr, sym) extern long __x64_##sym(const struct pt_regs *);
#include <asm/syscalls_64.h>
#undef __SYSCALL

#define __SYSCALL(nr, sym) __x64_##sym,

asmlinkage const sys_call_ptr_t sys_call_table[] = {
#include <asm/syscalls_64.h>
}
```

sys_call_ptr_t 是个函数指针类型声明, arch/x86/include/asm/syscall.h

```c
// pt_regs is used for ptrace to save register context if necessary (see arch/x86/include/asm/ptrace.h)
typedef long (*sys_call_ptr_t)(const struct pt_regs *);
```

系统调用表通过脚本生成：tools/perf/arch/x86/entry/syscalls/syscalltbl.sh

arch/x86/entry/syscalls/syscall_64.tbl 声明了64位的系统调用号和记录数组

宏展开之后，系统调用表会类似：

```c
extern long __x64_sys_read(const struct pt_regs *);
extern long __x64_sys_write(const struct pt_regs *);
...
extern long __x64_sys_ni_syscall(const struct pt_regs *);
...

asmlinkage const sys_call_ptr_t sys_call_table[] = {
    __x64_sys_read,
    __x64_sys_write,
    ...
    __x64_sys_ni_syscall,
    ...
};
```

## `SYSCALL_DEFINE0`

前面的宏定义部分提到，`SYSCALL_DEFINE0` 是一个较为特殊的宏。x86架构下，构建系统会选择 `ARCH_HAS_SYSCALL_WRAPPER` 宏，因此 `SYSCALL_DEFINE0` 会被前置替换掉，成为以下定义:

```c
#define SYSCALL_DEFINE0(sname)						\
    SYSCALL_METADATA(_##sname, 0);					\
    static long __do_sys_##sname(const struct pt_regs *__unused);	\
    __X64_SYS_STUB0(sname)						\
    __IA32_SYS_STUB0(sname)						\
    static long __do_sys_##sname(const struct pt_regs *__unused)
```

__X64_SYS_STUB0 定义在 arch/x86/include/asm/syscall_wrapper.h

```c
#define __X64_SYS_STUB0(name)						\
    __SYS_STUB0(x64, sys_##name)

#define __SYS_STUB0(abi, name)						\
    long __##abi##_##name(const struct pt_regs *regs);		\
    ALLOW_ERROR_INJECTION(__##abi##_##name, ERRNO);			\	// 需要宏 CONFIG_FUNCTION_ERROR_INJECTION，可以忽略
    long __##abi##_##name(const struct pt_regs *regs)		\
        __alias(__do_##name);
```

以 getpid 函数为例，上述封装会展开为：

```c
// SYSCALL_DEFINE0(getpid) {
//     return task_tgid_vnr(current);
// }

static long __do_sys_getpid(const struct pt_regs *__unused);
__X64_SYS_STUB0(getpid)
__SYS_STUB0(x64, sys_getpid)
    long __x64_sys_getpid(const struct pt_regs * regs);
    long __x64_sys_getpid(const struct pt_regs * regs) __alias(__do_sys_getpid);	// 与 DEFINEx 不同，直接别名
__IA32_SYS_STUB0(getpid)
static long __do_sys_getpid(const struct pt_regs *__unused) {
    return task_tgid_vnr(current);
}
```

## `SYSCALL_DEFINEn`

在 __SYSCALL_DEFINEx 中，包含 __MAP 和 __SC_xxx 两个宏，看起来主要是负责参数的传递：

```c
// example
__MAP(x,__SC_DECL,__VA_ARGS__)

/*
 * __MAP - apply a macro to syscall arguments
 * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
 *    m(t1, a1), m(t2, a2), ..., m(tn, an)
 * The first argument must be equal to the amount of type/name
 * pairs given.  Note that this list of pairs (i.e. the arguments
 * of __MAP starting at the third one) is in the same format as
 * for SYSCALL_DEFINE<n>/COMPAT_SYSCALL_DEFINE<n>
 */
#define __MAP0(m,...)
#define __MAP1(m,t,a,...) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)

#define __SC_DECL(t, a) t a
#define __TYPE_AS(t, v) __same_type((__force t) 0, v)
#define __TYPE_IS_LL(t) (__TYPE_AS(t, 0LL) || __TYPE_AS(t, 0ULL))
#define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
#define __SC_CAST(t, a) (__force t) a
#define __SC_ARGS(t, a) a
#define __SC_TEST(t, a) (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(t) && sizeof(t) > sizeof(long))

// include/linux/compiler_types.h
/* Are two types/vars the same type (ignoring qualifiers)? */
#define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))
```

## `__X64_SYS_STUBx`

```c
#define __X64_SYS_STUBx(x, name, ...) \
    __SYS_STUBx(x64, sys##name, \
        SC_X86_64_REGS_TO_ARGS(x, __VA_ARGS__))

#define __SYS_STUBx(abi, name, ...) \
    long __##abi##_##name(const struct pt_regs *regs); \
    ALLOW_ERROR_INJECTION(__##abi##_##name, ERRNO);    \
    long __##abi##_##name(const struct pt_regs *regs)  \
    { \
        return __se_##name(__VA_ARGS__); \
    }

/* Mapping of registers to parameters for syscalls on x86-64 and x32 */
#define SC_X86_64_REGS_TO_ARGS(x, ...) \
    __MAP(x,__SC_ARGS \
        ,,regs->di,,regs->si,,regs->dx \
        ,,regs->r10,,regs->r8,,regs->r9) \

```

## Example of expansion of `write`

```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
SYSCALL_DEFINEx(3, _write, unsigned int, fd, const char __user *, buf, size_t, count)
__SYSCALL_DEFINEx(3, _write, unsigned int, fd, const char __user *, buf, size_t, count)
```

```c
static long __se_sys_write(__MAP(3, __SC_LONG, unsigned int, fd, const char __user *, buf, size_t, count));
static long __se_sys_write(__SC_LONG(unsigned int, fd), __SC_LONG(const char __user *, buf), __SC_LONG(size_t, count));
    __SC_LONG(unsigned int, fd)
    __typeof(__builtin_choose_expr(__TYPE_IS_LL(unsigned int), 0LL, 0L)) fd
    __typeof(__builtin_choose_expr((__TYPE_AS(unsigned int, 0LL) || __TYPE_AS(unsigned int, 0ULL)), 0LL, 0L)) fd
    __typeof(__builtin_choose_expr(false, 0LL, 0L)) fd
    __typeof(0L) fd
    long fd
static long __se_sys_write(long fd, long long buf, long count);
```

```c
static inline long __do_sys_write(__MAP(3, __SC_DECL, unsigned int, fd, const char __user *, buf, size_t, count));
static inline long __do_sys_write(__SC_DECL(unsigned int, fd), __SC_DECL(const char __user *, buf), __SC_DECL(size_t, count));
static inline long __do_sys_write(unsigned int fd, const char __user * buf, size_t count);
```

```c
__X64_SYS_STUBx(3, write, unsigned int, fd, const char __user *, buf, size_t, count)
__SYS_STUBx(x64, sys_write, SC_X86_64_REGS_TO_ARGS(3, unsigned int, fd, const char __user *, buf, size_t, count))
__SYS_STUBx(x64, sys_write, __MAP(3, __SC_ARGS, , regs->di, , regs->si, , regs->dx, , regs->r10, , regs->r8, , regs->r9))
__SYS_STUBx(x64, sys_write, regs->di, regs->si, regs->dx)
    long __x64_sys_write(const struct pt_regs * regs);
    long __x64_sys_write(const struct pt_regs * regs)
    {
        return __se_sys_write(regs->di, regs->si, regs->dx);
    }
```

```c
static long __se_sys_write(long fd, long long buf, long count)
{
    long ret = __do_sys_write((__force unsigned int) fd, (__force const char __user *) buf, (__force size_t) count);
    __MAP(x, __SC_TEST, __VA_ARGS__); // we simply ignore it because we assume the tests are passed
    __PROTECT(x, ret, __MAP(x, __SC_ARGS, __VA_ARGS__)); // we simply ignore it because x86_64 doesn't use it
    return ret;
}
```

```c
static inline long __do_sys_write(unsigned int fd, const char __user * buf, size_t count) {
    return ksys_write(fd, buf, count);
}
ssize_t ksys_write(unsigned int fd, const char __user * buf, size_t count) {
    // ...
}
```

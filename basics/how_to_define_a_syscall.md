# 从源码理解Linux系统调用

笔者参考的是 linux v5.4 的源码，这个版本中通过 SYSCALL_DEFINEx() 宏来定义系统调用。

## 一个例子开始

如下是一个linux已经定义的系统调用例子，可以看到，在系统调用定义中，除了给出函数名外，函数的参数类型和参数名也应当依次给出，并使用逗号分隔。给出的参数类型和参数名，会作为可变模板参数，在后续的展开过程中，用于生成这个系统调用的元信息和函数结构。

```c
// /fs/read_write.c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count){
    ...
}
```
 
在头文件 `/include/linux/syscalls.h` 中：

* 可以看到目前支持最多6个参数的系统调用定义，这是因为大多数架构都不支持7个系统调用参数的传递；
* 一个系统调用包含元信息和函数结构
* 注意到这里的函数名被拼接了一个前缀'_'，主要是后续继续拼接时使用
* 对于零参数的系统调用定义，是直接给出的，没有依赖`SYSCALL_DEFINEx`

```c
#ifndef SYSCALL_DEFINE0
#define SYSCALL_DEFINE0(sname)                      \
    SYSCALL_METADATA(_##sname, 0);                  \
    asmlinkage long sys_##sname(void);              \
    ALLOW_ERROR_INJECTION(sys_##sname, ERRNO);      \
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
```

## 元信息部分

```c
#define SYSCALL_METADATA(sname, nb, ...)			\ 
    static const char *types_##sname[] = {			\
        __MAP(nb,__SC_STR_TDECL,__VA_ARGS__)		\
    };												\
    static const char *args_##sname[] = {			\
        __MAP(nb,__SC_STR_ADECL,__VA_ARGS__)		\
    };												\
    SYSCALL_TRACE_ENTER_EVENT(sname);				\
    SYSCALL_TRACE_EXIT_EVENT(sname);				\
    static struct syscall_metadata __used			\
      __syscall_meta_##sname = {					\
        .name 		= "sys"#sname,					\
        .syscall_nr	= -1,	/* Filled in at boot */	\
        .nb_args 	= nb,							\
        .types		= nb ? types_##sname : NULL,	\
        .args		= nb ? args_##sname : NULL,		\
        .enter_event	= &event_enter_##sname,		\
        .exit_event	= &event_exit_##sname,			\
        .enter_fields	= LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
    };												\
    static struct syscall_metadata __used			\
      __attribute__((section("__syscalls_metadata")))	\
     *__p_syscall_meta_##sname = &__syscall_meta_##sname;
```

宏的入参分别是：函数名、参数个数、参数

`__MAP` 宏类似于`python`中的`map`，即对列表中的每一个元素进行操作，`__SC_STR_TDECL` 用于在二元组中提取第一个作为类型，`__SC_STR_ADECL`用于在二元组中提取第二个作为参数名，因此元信息的前几行将参数列表中的参数类型和参数名字分别映射到`types_函数名[]`和`args_函数名[]`中，作为全局静态变量。

`SYSCALL_TRACE_ENTER_EVENT` 和 `SYSCALL_TRACE_EXIT_EVENT` 分别为这个函数生成了两个事件，用于在系统调用进入和退出时打印相关的信息。

然后生成了一个全局静态变量，用于保存这个系统调用的元信息，注意这里的函数名是 `sys_函数名`。

## 函数结构部分

```c
/*
 * The asmlinkage stub is aliased to a function named __se_sys_*() which
 * sign-extends 32-bit ints to longs whenever needed. The actual work is
 * done within __do_sys_*().
 */
#ifndef __SYSCALL_DEFINEx
#define __SYSCALL_DEFINEx(x, name, ...)								\
    __diag_push();													\
    __diag_ignore(GCC, 8, "-Wattribute-alias",						\
              "Type aliasing is used to sanitize syscall arguments");\
    asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))		\
        __attribute__((alias(__stringify(__se_sys##name))));		\
    ALLOW_ERROR_INJECTION(sys##name, ERRNO);						\
    static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
    asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
    asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
    {																\
        long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));	\
        __MAP(x,__SC_TEST,__VA_ARGS__);								\
        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));			\
        return ret;													\
    }																\
    __diag_pop();													\
    static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
#endif /* __SYSCALL_DEFINEx */
```

首先是声明需要插入新的诊断信息，这里在GCC_8版本中，忽略了"-Wattribute-alias"校验，因为这个函数的定义中，会使用`__attribute__((alias(__stringify(__se_sys##name))))`来定义一个新的函数，而这个函数的名字是`__se_sys_函数名`

接下来就是函数的声明，即系统调用为 `sys_函数名`，它具有 `__se_sys_函数名` 的别名，然后前置声明了`__do_sys_函数名` 函数，被 `__se_sys_函数名` 调用。

* 注意到，`__se_sys_函数名`的参数是使用`__SC_LONG`定义的, 因此在展开时会将参数类型转换为`long`。这里转换的目的，因为使用`long`型的版本可以在一些64位内核系统上使得在兼容32位应用时还可以正确的被转换。
  
  ```c
    #define __TYPE_AS(t, v)	__same_type((__force t)0, v)
    #define __TYPE_IS_L(t)	(__TYPE_AS(t, 0L))
    #define __TYPE_IS_UL(t)	(__TYPE_AS(t, 0UL))
    #define __TYPE_IS_LL(t) (__TYPE_AS(t, 0LL) || __TYPE_AS(t, 0ULL))
    #define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
  ```

在`__se_sys_函数名`中做了四件事：

1. 调用`__do_sys_函数名`，即实际的系统调用函数，并获得返回值
  * 在调用的时候，`#define __SC_CAST(t, a)	(__force t) a` 会对参数进行强制类型转换
2.  进行编译校验，即检查参数类型是否正确，
  * `#define __SC_TEST(t, a) (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(t) && sizeof(t) > sizeof(long))` `BUILD_BUG_ON_ZERO(e)`，在`e`为真的时候，会抛出一个编译错误
3. 注意到`__se_sys_函数名`使用了`asmlinkage`进行修饰，它会从栈上获取参数而不是寄存器上。调用的时候使用`#define __PROTECT(...) asmlinkage_protect(__VA_ARGS__)`进行修饰，可以防止编译器重用栈上的这块区域，否则修改了保存寄存器的栈区域，当系统调用结束的时候从栈上弹出寄存器内容的时候就会出现问题。
4. 最后返回系统调用的返回值

最后，这个宏展开生成了`__do_sys_函数名`的函数签名，衔接上了函数体。

## 外部调用

上面的展开我们还能看到，`__do_sys_函数名`是被声明为static的，即仅在当前编译单元可见。那外部如何调用呢？

外部调用的入口实际上是 `sys_函数名`，还是回到 `read` 的例子。

在 `/fs/read_write.c` 中完成展开后，函数已经完成了定义，其声明则是在 `/include/linux/syscalls.h` 中：

```c
asmlinkage long sys_read(unsigned int fd, char __user *buf, size_t count);
```

但是用户在使用系统调用的时候，并不会直接使用 `sys_read`，而是使用 `read`。在系统调用表`/arch/x86/entry/syscalls/syscall_64.tbl`中，我们可以找到 `read` 的系统调用号:

```tbl
#
# 64-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point>
#
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
#
# The abi is "common", "64" or "x32" for this file.
#
0	common	read			__x64_sys_read
...
```

在 linux v5.4 上实现的，是一种被称之为快速系统调用的技术，即[Fase system calls](https://www.felixcloutier.com/x86/syscall)，`SYSENTER` / `SYSCALL`，它们是 Intel 和 AMD 上用于实现快速系统调用的指令，在 32 位的操作系统上使用 `SYSENTER `/ `SYSEXIT`，在 64 位的操作系统上使用 `SYSCALL` / `SYSRET`。

内核在初始化时会调用 syscall_init 函数将 entry_SYSCALL_64 存入 MSR 寄存器（Model Specific Register、MSR）中，MSR 寄存器是 x86 指令集中用于调试、追踪以及性能监控的控制寄存器。

`SYSCALL`以特权级别0调用OS系统调用处理程序。它是通过从IA32_LSTAR MSR加载RIP来实现的（将SYSCALL之后的指令的地址保存到RCX中）。

```c
// /arch/x86/kernel/cpu/common.c
/* May not be marked __init: used by software suspend */
void syscall_init(void)
{
    wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
    wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
    ...
}
```

`MSR_LSTAR`在`arch/x86/include/uapi/asm/msr-index.h`中被定义`#define MSR_LSTAR		0xc0000082 /* long mode SYSCALL target */`

`entry_SYSCALL_64`是定义在`/arch/x86/entry/entry_64.S`中，通过汇编实现的方法，可以看到：

* syscall通过寄存器传递最多6个参数

```S
/*
 * 64-bit SYSCALL instruction entry. Up to 6 arguments in registers.
 *
 * This is the only entry point used for 64-bit system calls.  The
 * hardware interface is reasonably well designed and the register to
 * argument mapping Linux uses fits well with the registers that are
 * available when SYSCALL is used.
 *
 * SYSCALL instructions can be found inlined in libc implementations as
 * well as some other programs and libraries.  There are also a handful
 * of SYSCALL instructions in the vDSO used, for example, as a
 * clock_gettimeofday fallback.
 *
 * 64-bit SYSCALL saves rip to rcx, clears rflags.RF, then saves rflags to r11,
 * then loads new ss, cs, and rip from previously programmed MSRs.
 * rflags gets masked by a value from another MSR (so CLD and CLAC
 * are not needed). SYSCALL does not save anything on the stack
 * and does not change rsp.
 *
 * Registers on entry:
 * rax  system call number
 * rcx  return address
 * r11  saved rflags (note: r11 is callee-clobbered register in C ABI)
 * rdi  arg0
 * rsi  arg1
 * rdx  arg2
 * r10  arg3 (needs to be moved to rcx to conform to C ABI)
 * r8   arg4
 * r9   arg5
 * (note: r12-r15, rbp, rbx are callee-preserved in C ABI)
 *
 * Only called from user space.
 *
 * When user can change pt_regs->foo always force IRET. That is because
 * it deals with uncanonical addresses better. SYSRET has trouble
 * with them due to bugs in both AMD and Intel CPUs.
 */

ENTRY(entry_SYSCALL_64)
    ...
    call	do_syscall_64
    ...
```

## 深入阅读

- [The Definitive Guide to Linux System Calls](https://blog.packagecloud.io/the-definitive-guide-to-linux-system-calls/)
- [为什么系统调用会消耗较多资源？系统调用的三种方法：软件中断（分析过程）、SYSCALL指令、vDSO(虚拟动态链接对象linux-vdso.so.1)](https://rtoax.blog.csdn.net/article/details/109145095)
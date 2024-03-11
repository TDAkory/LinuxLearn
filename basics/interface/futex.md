# [futex](https://en.wikipedia.org/wiki/Futex#:~:text=In%20computing%2C%20a%20futex%20(short,POSIX%20mutexes%20or%20condition%20variables.)

> In computing, a futex (short for "fast userspace mutex") is a kernel system call that programmers can use to implement basic locking, or as a building block for higher-level locking abstractions such as semaphores and POSIX mutexes or condition variables.

- [futex(2) — Linux manual page](https://man7.org/linux/man-pages/man2/futex.2.html)

```c
#include <linux/futex.h>      /* Definition of FUTEX_* constants */
#include <sys/syscall.h>      /* Definition of SYS_* constants */
#include <unistd.h>

long syscall(SYS_futex, uint32_t *uaddr, int futex_op, uint32_t val,
             const struct timespec *timeout,   /* or: uint32_t val2 */
             uint32_t *uaddr2, uint32_t val3);
```

- futex可以再用户态跨进程使用，也可以在进程内跨线程使用。如果仅仅是线程同步场景，相比于进程同步，内核对原子操作还有额外的优化，因此提供了额外的参数 XXX_PRIVATE。

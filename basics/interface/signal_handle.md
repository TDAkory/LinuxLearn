## Singal process

信号处理是 Unix/Linux 系统编程的核心，用于处理异步事件。下面我将为你详细解释常用的 C API，包括其含义、用法和简洁的示例。

### 核心概念与类型

1. **信号**：信号是系统或进程发送给另一个进程的简短异步通知，用于告知某事件的发生（如 `SIGINT` 对应 Ctrl+C 中断，`SIGSEGV` 对应段错误）。
2. **信号集**：`sigset_t` 类型用于表示一组信号的集合。它是后续所有信号操作（如阻塞、等待）的基础容器。

### 关键 API 详解与示例

#### **1. 信号集操作**

这些函数用于操作 `sigset_t` 类型的信号集合。

**`sigemptyset` / `sigfillset`**：初始化信号集。`sigemptyset` 清空信号集（不包含任何信号）；`sigfillset` 填充所有信号。
  
```c
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
```

**用法示例**：

```c
sigset_t set;
sigemptyset(&set); // 初始化一个空集
sigaddset(&set, SIGINT); // 然后向其中添加信号

sigset_t all_signals;
sigfillset(&all_signals); // 初始化包含所有信号的集合
```

**`sigaddset` / `sigdelset` / `sigismember`**：向集合中添加 (`sigaddset`) 或删除 (`sigdelset`) 特定信号。`sigismember` 用于检查一个信号是否在集合中。

```c
int sigaddset(sigset_t *set, int signum);
int sigdelset(sigset_t *set, int signum);
int sigismember(const sigset_t *set, int signum);
```

#### **2. 信号处理函数设置**

**`signal`**：为指定信号安装一个简单的处理函数。**注意**：`signal` 的行为在不同Unix版本有差异，对于新程序，建议使用更强大、行为一致的 `sigaction`。

`void (*signal(int signum, void (*handler)(int)))(int);`

**用法示例**：

```c
#include <stdio.h>
#include <signal.h>
void handler(int sig) {
    printf("Caught signal %d\n", sig);
}
int main() {
    signal(SIGINT, handler); // 为SIGINT安装自定义处理函数
    while(1);
    return 0;
}
```

**`sigaction` (强烈推荐替代`signal`)**：提供了更精细的控制，用于检查或设置信号的处理动作。

```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

**用法示例**：

```c
struct sigaction sa;
sa.sa_handler = handler; // 设置处理函数
sigemptyset(&sa.sa_mask); // 在处理函数执行期间，不额外阻塞其他信号
sa.sa_flags = 0;
sigaction(SIGINT, &sa, NULL); // 安装
```

#### **3. 信号屏蔽（阻塞）**

用于控制进程或线程在哪些时段不响应哪些信号。被阻塞的信号会处于“未决”状态，直到解除阻塞。

**`pthread_sigmask` (多线程环境使用)**：设置或获取**当前线程**的信号掩码（即阻塞哪些信号）。这是多线程程序中管理信号的关键函数。

```c
int pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset);
```

**参数`how`**：

* `SIG_BLOCK`：将 `set` 中的信号添加到当前阻塞信号集中。
* `SIG_UNBLOCK`：从当前阻塞集中移除 `set` 中的信号。
* `SIG_SETMASK`：将当前阻塞集直接设置为 `set`。

**用法示例**：

```c
sigset_t block_set;
sigemptyset(&block_set);
sigaddset(&block_set, SIGUSR1);
// 在主线程中阻塞SIGUSR1，确保只有工作线程能处理它
pthread_sigmask(SIG_BLOCK, &block_set, NULL);
```

**`sigprocmask` (单线程/主线程环境使用)**：用于**单线程进程**或**多线程进程的主线程**中设置或获取进程的信号掩码。在子线程中使用行为未定义。

`int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);`

#### **4. 同步等待信号**

**`sigwait`**：**同步地**等待并接收一个被阻塞的信号。它会挂起调用线程，直到指定的信号集中的任一信号到达。它**会消费掉这个未决信号**，使其不再递送。

`int sigwait(const sigset_t *set, int *sig);`

**用法示例**：

```c
void* thread_func(void* arg) {
    sigset_t wait_set;
    sigfillset(&wait_set); // 本线程等待所有信号
    int sig_received;
    while(1) {
        sigwait(&wait_set, &sig_received); // 阻塞，直到收到信号
        printf("Thread received signal: %d\n", sig_received);
        // 在此进行安全的信号处理（注意：这是在普通的线程上下文中，不是异步处理函数）
    }
    return NULL;
}
```

### 多线程信号处理最佳实践模式

一个健壮的、利用上述API的多线程程序信号处理架构通常如下：

1. **主线程屏蔽所有信号**：在创建任何子线程**之前**，使用 `pthread_sigmask` 阻塞所有你感兴趣的信号。这样，所有子线程都会继承这个屏蔽集，保证信号不会被异步递送给任何线程。
2. **创建专有信号处理线程**：创建一个或多个专用线程，它们使用 `sigwait` 来同步等待并处理被阻塞的信号。这样就将异步信号“转换”为了同步事件，避免了在异步处理函数中调用非异步安全函数带来的限制和风险。

```c
#include <pthread.h>
#include <signal.h>
#include <stdio.h>

void* signal_handler_thread(void* arg) {
    sigset_t set;
    int sig;
    sigfillset(&set); // 等待所有信号
    while(1) {
        sigwait(&set, &sig);
        switch(sig) {
            case SIGINT:
                printf(“[Handler Thread] Graceful shutdown initiated.\n”);
                // 执行安全的清理操作...
                break;
            case SIGUSR1:
                // 处理用户自定义信号
                break;
        }
    }
    return NULL;
}

int main() {
    sigset_t all_signals;
    sigfillset(&all_signals);
    // 1. 主线程阻塞所有信号
    pthread_sigmask(SIG_BLOCK, &all_signals, NULL);
    
    // 2. 创建专有信号处理线程
    pthread_t handler_tid;
    pthread_create(&handler_tid, NULL, signal_handler_thread, NULL);
    
    // 3. 主程序逻辑继续运行，不受异步信号干扰
    while(1) {
        /* 你的主要工作逻辑 */
    }
    return 0;
}
```

### 其他常用API补充

* **`kill` / `pthread_kill`**：发送信号。`kill(pid_t pid, int sig)` 向进程发送；`pthread_kill(pthread_t thread, int sig)` 向特定线程发送。
* **`raise`**：向**自身**发送信号。`raise(sig)` 等价于 `pthread_kill(pthread_self(), sig)`。
* **`sigpending`**：获取当前进程（或线程）中哪些信号处于“未决”状态（即已产生但被阻塞）。

## Signal Handle

Linux为信号处理程序（handler）提供了三种标准处理方式，你可以通过 `sigaction` 结构体中的 `sa_handler` 字段或 `signal` 函数的第二个参数来指定。此外，用户还可以自定义函数作为第四种方式。它们通常被定义为“指向返回void且接受一个int参数（信号编号）的函数指针”类型。

### 三种标准处理方式

这三种方式本质上是预定义的宏，它们代表了由内核直接处理的特殊操作。

| 处理方式 | 含义与用途 | 使用场景示例 |
| :--- | :--- | :--- |
| **`SIG_DFL`** | **默认处理**。内核对此信号的**默认操作**，大多数信号（如 `SIGKILL`, `SIGTERM`, `SIGSEGV`）会导致进程终止，但有些信号（如 `SIGCHLD`、`SIGURG`）默认是忽略。 | 恢复某个信号的默认行为。例如，子进程继承了父进程对 `SIGINT` 的忽略，你可以显式设置为 `SIG_DFL` 来让 Ctrl-C 能终止它。 |
| **`SIG_IGN`** | **忽略信号**。内核收到此信号后，直接**丢弃它**，进程不会有任何感知。**注意**：`SIGKILL` 和 `SIGSTOP` **不能被忽略或捕获**。 | 忽略 `SIGPIPE` 信号，防止向已关闭的管道/套接字写数据时导致进程意外退出，改由 `write` 函数返回错误。 |
| **自定义函数** | **捕获信号**。指定一个**用户自定义的函数**，当信号递送时，内核会调用此函数来处理。 | 在自定义函数中设置退出标志位，实现程序的优雅终止。 |

### 示例

**`SIG_DFL` (默认处理)**

```c
#include <signal.h>
#include <stdio.h>
int main() {
    // 假设之前设置了忽略SIGINT
    signal(SIGINT, SIG_IGN);
    // 现在恢复为默认行为（即Ctrl-C会终止进程）
    signal(SIGINT, SIG_DFL);
    while(1);
    return 0;
}
```

**`SIG_IGN` (忽略信号)**

```c
#include <signal.h>
#include <unistd.h>
#include <stdio.h>
int main() {
    // 忽略 SIGPIPE 信号
    signal(SIGPIPE, SIG_IGN);
    // 现在如果发生管道写破裂，write()会返回-1并设置errno为EPIPE，
    // 而不是导致进程被信号终止。
    int fd[2];
    pipe(fd);
    close(fd[0]); // 关闭读端
    write(fd[1], "hello", 5); // 不会引发SIGPIPE信号杀死进程，而是返回错误
    perror("write");
    return 0;
}
```

**自定义处理函数**

```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>
// 自定义信号处理函数
void my_handler(int signo) {
    if (signo == SIGINT) {
        printf(“\n[自定义处理] 收到SIGINT，启动优雅退出流程…\n”);
        // 此处应只设置标志位，避免调用非异步安全函数（如printf）
    }
}
int main() {
    // 推荐使用sigaction，它比signal()更可控
    struct sigaction sa;
    sa.sa_handler = my_handler; // 设置自定义处理函数
    sigemptyset(&sa.sa_mask); // 在处理函数执行期间，不额外阻塞其他信号
    sa.sa_flags = 0; // 通常设置为0
    sigaction(SIGINT, &sa, NULL); // 捕获SIGINT
    printf(“程序已启动。按Ctrl-C测试自定义处理函数。\n”);
    while(1) {
        pause(); // 挂起进程，等待信号
    }
    return 0;
}
```

### 3. 特殊说明：`SIG_ERR`

* `SIG_ERR` 不是一个处理程序，而是 `signal()` 函数在**设置失败时返回的值**。它是一个类型为 `void (*)(int)` 的函数指针常量，通常被强制转换为 `(void (*)(int))-1`。

```c
// 用来检查 `signal()` 调用是否成功
if (signal(SIGINT, my_handler) == SIG_ERR) {
    perror(“signal”); // 设置信号处理函数失败
}
```

## 关键要点总结

* **优先使用`sigaction`而非`signal`**，以获得确定性和更强的控制力。
* **多线程环境中**，使用 `pthread_sigmask` 和 `sigwait` 的组合是处理信号的**黄金标准**，它能将异步信号安全地转化为同步事件。
* **避免在异步信号处理函数**（由 `signal` 或 `sigaction` 的 `sa_handler` 指定）中进行复杂操作或调用非异步安全函数（如 `printf`, `malloc`）。`sigwait` 模式则无此限制。
* **handle选择策略**：
  * 若想恢复内核默认行为 → **`SIG_DFL`**
  * 若想完全无视某个信号（针对可忽略的信号） → **`SIG_IGN`**
  * 若想执行自定义逻辑 → **自定义函数**
* **自定义函数安全警告**：在自定义信号处理函数中，**只能调用“异步信号安全”的函数**（如 `write`, `_exit`, `sigprocmask`）。像 `printf`, `malloc`, `free` 等标准I/O或内存函数是**不安全**的，可能引发死锁或未定义行为。这是推荐使用 `sigwait` 进行同步处理的核心原因。
* **不可更改的信号**：`SIGKILL` 和 `SIGSTOP` 是超级管理员信号，任何进程都无法更改其处理方式（即不能设置为 `SIG_IGN` 或自定义处理函数），确保了系统始终能强制终止或暂停一个进程。

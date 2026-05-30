# POSIX 进程控制

> 本文涵盖 POSIX 标准中进程控制相关的 C API：`fork`、`exec` 系列、`wait` 系列、`_exit`。

See also: [[POSIX进程派生]], [[POSIX用户组与环境]], [[C++POSIX文件操作]], [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]], [[POSIX定时器与时钟]], [[POSIX终端IO]]

---

## 一、进程创建

```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);                    // [POSIX]
pid_t vfork(void);                   // [POSIX] 已在POSIX.1-2008中移除，不推荐使用
```

**`fork` 核心要点**：
- 创建一个子进程，子进程是父进程的**副本**（代码段、数据段、堆、栈等）。
- 父进程中返回子进程 PID，子进程中返回 0，出错返回 -1。
- 现代系统采用**写时复制（Copy-On-Write, COW）**优化：父子进程共享内存页，仅当一方修改时才复制。
- 子进程继承父进程的文件描述符表，但父子进程共享同一文件偏移量。

**`vfork` 与 `fork` 的区别**：
- `vfork` 保证子进程先运行，父进程阻塞直到子进程调用 `exec` 或 `_exit`。
- 子进程共享父进程的地址空间（不复制），因此子进程对数据的修改会影响父进程。
- **已在 POSIX.1-2008 中移除**，不推荐使用。

---

## 二、进程执行（exec 系列）

```c
#include <unistd.h>

int execl(const char *path, const char *arg, ...);                   // [POSIX]
int execlp(const char *file, const char *arg, ...);                  // [POSIX]
int execv(const char *path, char *const argv[]);                     // [POSIX]
int execvp(const char *file, char *const argv[]);                    // [POSIX]
int execle(const char *path, const char *arg, ..., char *const envp[]); // [POSIX]
int execve(const char *path, char *const argv[], char *const envp[]);   // [POSIX]
int fexecve(int fd, char *const argv[], char *const envp[]);            // [POSIX.1-2008]
```

**命名规则记忆**：
- `l`（list）：参数以可变参数列表形式传递，以 `NULL` 结尾。
- `v`（vector）：参数以 `char*` 数组形式传递。
- `p`（path）：在 `PATH` 环境变量中搜索可执行文件。
- `e`（environment）：可指定新的环境变量数组。
- `f`（file descriptor）：通过文件描述符指定可执行文件（`fexecve`）。

**核心要点**：
- `exec` 系列函数用新程序**替换**当前进程的地址空间，成功时不返回，失败返回 -1。
- `execve` 是真正的系统调用，其他 `exec` 函数都是对 `execve` 的封装。
- 调用 `exec` 后，原进程的文件描述符默认保留（除非设置了 `FD_CLOEXEC` 标志）。

---

## 三、进程等待

```c
#include <sys/wait.h>

pid_t wait(int *status);                            // [POSIX]
pid_t waitpid(pid_t pid, int *status, int options);  // [POSIX]
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options); // [POSIX]
```

**`wait` vs `waitpid`**：
- `wait`：阻塞等待**任意**一个子进程终止，返回终止子进程的 PID。
- `waitpid`：可指定等待特定 PID 的子进程，支持非阻塞选项（`WNOHANG`）。

**`waitpid` 的 `pid` 参数**：
- `pid > 0`：等待指定 PID 的子进程。
- `pid == 0`：等待与调用进程同进程组的任意子进程。
- `pid == -1`：等待任意子进程（等价于 `wait`）。
- `pid < -1`：等待进程组 ID 为 `|pid|` 的任意子进程。

**`status` 解析宏**：
- `WIFEXITED(status)`：正常终止时返回真，`WEXITSTATUS(status)` 获取退出码。
- `WIFSIGNALED(status)`：被信号终止时返回真，`WTERMSIG(status)` 获取信号编号。
- `WIFSTOPPED(status)`：被暂停时返回真，`WSTOPSIG(status)` 获取信号编号。
- `WIFCONTINUED(status)`：被恢复执行时返回真。

**`options` 常用值**：
- `WNOHANG`：非阻塞，若无子进程终止则立即返回 0。
- `WUNTRACED`：同时报告被暂停的子进程。
- `WCONTINUED`：同时报告被恢复执行的子进程。

---

## 四、进程终止

```c
#include <unistd.h>

void _exit(int status);             // [POSIX]
```

**`_exit` vs `exit`**：
- `_exit`：**直接**终止进程，不执行任何清理操作（不调用 `atexit` 注册的函数、不刷新 stdio 缓冲区）。
- `exit`：标准 C 库函数，会调用 `atexit` 注册的函数、刷新 stdio 缓冲区、关闭文件流，最后调用 `_exit`。
- `fork` 后的子进程若需立即退出，应使用 `_exit` 避免重复执行父进程的 `atexit` 回调。

---

## 五、面试高频问题

### Q1: `fork` 后父子进程共享哪些资源？
- **共享**：打开的文件描述符（及文件偏移量）、内存映射（`mmap` 的 `MAP_SHARED`）、信号处理设置。
- **不共享**：地址空间（COW 后各自独立）、锁（`pthread` 锁状态不确定）、挂起的信号集。

### Q2: `fork` 后子进程继承了父进程的哪些属性？
- 实际/有效 UID/GID、进程组 ID、会话 ID、控制终端、当前工作目录、文件模式创建掩码（umask）、信号屏蔽字、环境变量、资源限制。

### Q3: `fork` + `exec` 的常见陷阱？
- 子进程继承了父进程的文件描述符，若未关闭可能导致资源泄漏或文件锁未释放。
- 多线程程序中 `fork` 后只有调用 `fork` 的线程在子进程中存在，其他线程消失，可能导致锁状态不一致。
- 应使用 `pthread_atfork` 注册回调处理锁状态。

### Q4: 如何避免僵尸进程？
- 父进程调用 `wait` / `waitpid` 回收子进程。
- 父进程显式忽略 `SIGCHLD` 信号：`signal(SIGCHLD, SIG_IGN)`。
- 将子进程的父进程改为 init 进程（两次 `fork`，让中间进程立即退出）。

### Q5: `exec` 系列函数执行后，原进程的哪些属性会保留？
- 进程 ID、父进程 ID、进程组 ID、会话 ID、实际 UID/GID、文件描述符（除非设置了 `FD_CLOEXEC`）、当前工作目录、资源限制、信号屏蔽字。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-2.-进程控制.md]

## Related Pages
- [[POSIX进程派生]]
- [[POSIX用户组与环境]]
- [[C++POSIX文件操作]]
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[POSIX定时器与时钟]]
- [[POSIX终端IO]]
- [[Linux线程调度]]

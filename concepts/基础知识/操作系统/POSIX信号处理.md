# POSIX 信号处理

> 本文涵盖 POSIX 标准中信号处理相关的 C API：信号发送、信号处理、信号集操作、信号阻塞与等待。

See also: [[POSIX进程控制]], [[POSIX进程派生]], [[POSIX定时器与时钟]], [[C++多线程与并发]]

---

## 一、信号发送

```c
#include <signal.h>
#include <unistd.h>

int kill(pid_t pid, int sig);                // [POSIX]
int raise(int sig);                          // [C/POSIX]
unsigned int alarm(unsigned int seconds);    // [POSIX]
int killpg(pid_t pgrp, int sig);             // [POSIX]
```

**API 说明**：
- `kill(pid, sig)`：向指定进程或进程组发送信号。`pid > 0` 发送给指定进程；`pid == 0` 发送给同进程组所有进程；`pid == -1` 发送给所有有权限的进程；`pid < -1` 发送给进程组 ID 为 `-pid` 的所有进程。
- `raise(sig)`：向当前进程发送信号，等价于 `kill(getpid(), sig)`。
- `alarm(seconds)`：设置闹钟，`seconds` 秒后向当前进程发送 `SIGALRM`。若 `seconds == 0` 则取消之前的闹钟。返回值是前一个闹钟剩余的秒数。
- `killpg(pgrp, sig)`：向指定进程组发送信号，等价于 `kill(-pgrp, sig)`。

---

## 二、信号处理

```c
sighandler_t signal(int signum, sighandler_t handler);                       // [C/POSIX]
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact); // [POSIX]
```

**API 说明**：
- `signal(signum, handler)`：设置信号 `signum` 的处理函数为 `handler`。`handler` 可以是 `SIG_IGN`（忽略）、`SIG_DFL`（默认处理）或自定义函数指针。**注意**：`signal` 在不同 UNIX 系统上行为不一致（BSD 语义 vs System V 语义），推荐使用 `sigaction`。
- `sigaction(signum, act, oldact)`：检查或修改信号 `signum` 的处理动作。`struct sigaction` 包含：
  - `sa_handler` / `sa_sigaction`：信号处理函数
  - `sa_mask`：处理信号期间需要阻塞的信号集
  - `sa_flags`：标志位（如 `SA_RESTART` 自动重启被中断的系统调用、`SA_SIGINFO` 使用扩展处理函数等）

**`signal` vs `sigaction`**：
- `signal` 是早期 C 标准函数，行为不可移植。
- `sigaction` 是 POSIX 标准推荐的信号处理方式，提供更精细的控制（信号屏蔽、重启语义、额外信息）。

---

## 三、信号集操作

```c
int sigemptyset(sigset_t *set);              // [POSIX]
int sigfillset(sigset_t *set);               // [POSIX]
int sigaddset(sigset_t *set, int signum);    // [POSIX]
int sigdelset(sigset_t *set, int signum);    // [POSIX]
int sigismember(const sigset_t *set, int signum); // [POSIX]
```

**API 说明**：
- `sigemptyset(set)`：初始化信号集为空。
- `sigfillset(set)`：初始化信号集包含所有信号。
- `sigaddset(set, signum)`：向信号集中添加指定信号。
- `sigdelset(set, signum)`：从信号集中删除指定信号。
- `sigismember(set, signum)`：测试指定信号是否在信号集中。

**注意**：`sigset_t` 在使用前必须用 `sigemptyset` 或 `sigfillset` 初始化。

---

## 四、信号阻塞与等待

```c
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);   // [POSIX]
int sigpending(sigset_t *set);               // [POSIX]
int sigsuspend(const sigset_t *mask);        // [POSIX]
int sigwait(const sigset_t *set, int *sig);  // [POSIX]
int sigwaitinfo(const sigset_t *set, siginfo_t *info);              // [POSIX]
int sigtimedwait(const sigset_t *set, siginfo_t *info,
                 const struct timespec *timeout);                    // [POSIX]
int pause(void);                             // [POSIX]
```

**API 说明**：
- `sigprocmask(how, set, oldset)`：检查或修改当前线程的信号屏蔽字。`how` 参数：
  - `SIG_BLOCK`：将 `set` 中的信号添加到屏蔽字（并集）
  - `SIG_UNBLOCK`：从屏蔽字中移除 `set` 中的信号
  - `SIG_SETMASK`：将屏蔽字设置为 `set`
- `sigpending(set)`：获取当前线程阻塞且未决（pending）的信号集。
- `sigsuspend(mask)`：原子地将当前线程的信号屏蔽字替换为 `mask`，然后挂起等待信号。收到信号并执行完处理函数后返回，同时恢复原屏蔽字。
- `sigwait(set, sig)`：同步等待 `set` 中的任一信号到达，将信号编号存入 `*sig`，不执行信号处理函数。
- `sigwaitinfo(set, info)`：类似 `sigwait`，但返回更多信号信息（`siginfo_t`）。
- `sigtimedwait(set, info, timeout)`：带超时的 `sigwaitinfo`。
- `pause()`：挂起当前线程直到收到一个信号（该信号需有处理函数或终止进程）。

**`sigsuspend` vs `sigwait`**：
- `sigsuspend`：信号到达后执行信号处理函数，然后返回。
- `sigwait`：信号到达后不执行处理函数，直接返回信号编号。适合多线程程序中用专门线程同步处理信号。

---

## 五、面试高频问题

### Q1: `signal` 和 `sigaction` 的区别？
- `signal` 是早期 C 标准函数，不同 UNIX 系统行为不一致（BSD 语义：信号处理函数执行期间自动屏蔽该信号，处理完后恢复；System V 语义：处理一次后恢复默认行为）。
- `sigaction` 是 POSIX 标准推荐的替代方案，提供更精细的控制：可指定处理期间屏蔽的信号集、是否自动重启被中断的系统调用（`SA_RESTART`）、获取发送信号的进程信息（`SA_SIGINFO`）等。
- **面试建议**：始终推荐使用 `sigaction`。

### Q2: `sigsuspend` 的典型用途？
- 原子地解除信号屏蔽并挂起等待，避免 `sigprocmask` + `pause` 之间的竞态条件（信号可能在 `pause` 之前到达）。
- 常用于实现信号驱动的同步等待。

### Q3: 多线程程序中如何处理信号？
- 信号处理是进程级别的，任意线程都可能收到信号。
- 推荐做法：在主线程中屏蔽所有信号（`sigprocmask`），然后创建一个专用线程调用 `sigwait` / `sigwaitinfo` 同步等待并处理信号。
- 避免在信号处理函数中做复杂操作（异步信号安全限制）。

### Q4: `kill(pid, 0)` 的作用？
- 发送信号 0（空信号）用于**错误检查**：若 `kill` 返回 0 且 `errno` 为 `ESRCH`，则目标进程不存在；否则进程存在且有权限。常用于检测进程是否存活。

### Q5: `alarm` 和 `setitimer` 的区别？
- `alarm` 只能设置秒级精度的单次定时器，到期发送 `SIGALRM`。
- `setitimer` 提供微秒级精度，支持周期定时，可发送 `SIGALRM`、`SIGVTALRM`、`SIGPROF` 等不同信号。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-3.-信号处理.md]

## Related Pages
- [[POSIX进程控制]]
- [[POSIX进程派生]]
- [[POSIX定时器与时钟]]
- [[C++多线程与并发]]
- [[Linux线程调度]]

# 系统编程与 IPC

> POSIX API、Linux 系统调用、进程管理、进程间通信（管道/共享内存/信号量/信号/Socket）。

---

## 一、进程管理

### 1.1 fork + exec —— Linux 创建新进程的标准两步操作

**fork()** 复制当前进程（写时复制 COW），**exec()** 用新程序替换子进程映像。
两步分开的原因是：fork 后 exec 前可以做一些准备工作（重定向 fd、设置环境变量、切换用户等），
这是 shell 管道 `|` 和 I/O 重定向的实现基础。

```cpp
#include <unistd.h>
#include <sys/wait.h>

pid_t pid = fork();
if (pid < 0) { /* 错误处理 */ }
if (pid == 0) {
    // ── 子进程 ──
    // 可以在这里做准备工作：重定向 fd、设置环境变量、切换用户等
    dup2(pipefd[1], STDOUT_FILENO);   // 示例：将标准输出重定向到管道写端
    close(pipefd[0]); close(pipefd[1]);

    execl("/bin/echo", "echo", "hello", nullptr);  // 替换为 echo 程序
    _exit(1);  // exec 失败才执行到这里
} else {
    // ── 父进程 ──
    wait(nullptr);  // 等待子进程结束，回收资源
}
```

#### 关键点详解

| 概念 | 说明 |
|------|------|
| **fork() 返回值** | 子进程返回 0，父进程返回子进程 PID，失败返回 -1 |
| **写时复制 COW** | fork 后父子进程共享物理内存页，只有写入时才复制，减少 fork 开销 |
| **fork 后共享** | 代码段（只读）、文件描述符（指向同一偏移量）、mmap(MAP_SHARED) |
| **fork 后私有** | 数据段（COW）、栈、堆、信号处理器、信号掩码 |
| **exec 族函数** | `execl`/`execlp`/`execle`/`execv`/`execvp`/`execvpe`（l=list, v=vector, p=PATH搜索, e=环境变量） |
| **exec 后的保留** | pid、ppid、fd（除非设 FD_CLOEXEC）、当前工作目录、信号处理器（继承自忽略/默认，自定义行为重置） |
| **exec 后的重置** | 地址空间完全替换（数据段、堆、栈全部丢弃）、信号处理器（自定义→默认）、atexit 回调 |

#### exit  vs  _exit

```cpp
exit(0);   // C 库函数：刷新 stdio 缓冲区 → 执行 atexit 回调 → 调用全局对象析构 → _exit()
_exit(0);  // 系统调用：直接内核终止进程，不执行任何清理
// 子进程 fork 后必须用 _exit()，否则会误刷父进程的 stdio 缓冲区，造成数据混乱
```

#### 典型应用：shell 管道

```cpp
// 模拟 shell 命令：echo hello | wc -c
int pipefd[2];
pipe(pipefd);                     // 创建管道

pid_t pid = fork();
if (pid == 0) {
    // 子进程：echo（写端）
    close(pipefd[0]);             // 关闭读端
    dup2(pipefd[1], STDOUT_FILENO); // 标准输出重定向到管道写端
    close(pipefd[1]);
    execlp("echo", "echo", "hello", nullptr);
    _exit(1);
}

pid_t pid2 = fork();
if (pid2 == 0) {
    // 子进程2：wc（读端）
    close(pipefd[1]);             // 关闭写端
    dup2(pipefd[0], STDIN_FILENO);  // 标准输入从管道读端重定向
    close(pipefd[0]);
    execlp("wc", "wc", "-c", nullptr);
    _exit(1);
}

close(pipefd[0]); close(pipefd[1]);  // 父进程关闭管道两端
wait(nullptr); wait(nullptr);        // 等待两个子进程
```

### 1.2 进程生命周期

#### 进程状态（Linux）

| 状态 | 内核标识 | 说明 |
|:----:|:--------:|------|
| **运行** | R (TASK_RUNNING) | 正在运行或可运行队列中等待调度 |
| **可中断睡眠** | S (TASK_INTERRUPTIBLE) | 等待某事件（如 I/O 完成），可被信号唤醒 |
| **不可中断睡眠** | D (TASK_UNINTERRUPTIBLE) | 等待内核 I/O 完成，不可被信号打断（如磁盘读写），kill -9 也杀不掉 |
| **停止** | T (TASK_STOPPED) | 收到 SIGSTOP/SIGTSTP 等信号暂停 |
| **僵尸** | Z (TASK_ZOMBIE) | 子进程已退出但父进程未 wait，仅保留进程表项 |
| **死亡** | X (TASK_DEAD) | 即将被销毁，几乎不可见 |

```bash
# 查看进程状态
ps aux          # STAT 列：R/S/D/T/Z/X
ps -eo pid,stat,wchan:32,comm   # 加 wchan 显示阻塞在内核哪个函数
```

#### 进程回收

```cpp
pid_t wait(int* status);                      // 阻塞等待任意子进程
pid_t waitpid(pid_t pid, int* status, int options);  // 等待指定子进程
// options: WNOHANG（非阻塞）, WUNTRACED（停止也返回）

// 状态宏
WIFEXITED(status)     // 正常退出 → WEXITSTATUS(status) 获取退出码
WIFSIGNALED(status)   // 被信号终止 → WTERMSIG(status) 获取终止信号
WIFSTOPPED(status)    // 被暂停 → WSTOPSIG(status) 获取暂停信号
```

#### 孤儿进程 vs 僵尸进程

| 类型 | 产生条件 | 处理方式 |
|:----:|----------|----------|
| **孤儿进程** | 父进程先退出，子进程还在运行 | init（pid=1）收养，自动 wait 回收 |
| **僵尸进程** | 子进程先退出，父进程未 wait | 占进程表项（kill -9 杀不掉），父进程必须 wait |

**防止僵尸进程的三种方法**：

```cpp
// 方法 1：signal(SIGCHLD, SIG_IGN) —— 内核自动回收
signal(SIGCHLD, SIG_IGN);

// 方法 2：显式 waitpid 循环（推荐）
while (true) {
    pid_t pid = waitpid(-1, &status, WNOHANG);
    if (pid <= 0) break;  // 没有已退出的子进程
}

// 方法 3：双重 fork —— 子进程立即退出，由 init 回收
pid_t pid = fork();
if (pid == 0) {
    pid_t pid2 = fork();  // 孙进程
    if (pid2 == 0) {
        // 孙进程执行业务逻辑
    }
    _exit(0);  // 子进程立即退出，父进程 wait 回收
}
wait(nullptr);  // 父进程回收子进程
```

---

## 二、进程间通信（IPC）

### 2.1 管道

```cpp
// 匿名管道（父子进程）
int pipefd[2];
pipe(pipefd);  // pipefd[0] 读端，pipefd[1] 写端

// 典型用法：fork 后父子进程各关一端，形成单向数据流
pid_t pid = fork();
if (pid == 0) {
    close(pipefd[0]);                // 子进程关读端
    write(pipefd[1], "hello", 5);
    close(pipefd[1]);
    _exit(0);
} else {
    close(pipefd[1]);                // 父进程关写端
    char buf[64];
    read(pipefd[0], buf, sizeof(buf));
    close(pipefd[0]);
    wait(nullptr);
}

// 命名管道（任意进程，文件系统可见）
mkfifo("/tmp/myfifo", 0666);
// 进程A（写）：int fd = open("/tmp/myfifo", O_WRONLY); write(fd, data, len);
// 进程B（读）：int fd = open("/tmp/myfifo", O_RDONLY); read(fd, buf, len);

// 双向通信需两个管道（半双工限制）
int pipe_parent[2], pipe_child[2];
pipe(pipe_parent); pipe(pipe_child);
// 父进程：pipe_parent 写 → 子进程读；pipe_child 读 ← 子进程写
```

### 2.2 共享内存

```cpp
#include <sys/mman.h>

// POSIX 共享内存（任意进程）
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void* ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// 匿名共享内存（亲缘进程，无需文件描述符）
void* ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);

// 父子进程通过共享内存通信
pid_t pid = fork();
int* counter = (int*)mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE,
                          MAP_SHARED | MAP_ANONYMOUS, -1, 0);
if (pid == 0) {
    (*counter)++;   // 父进程可见
    _exit(0);
} else {
    wait(nullptr);
    printf("%d\n", *counter);  // 1
}
```

**⚠️ 共享内存必须配合同步原语（mutex、信号量）使用**，否则产生数据竞争。
Linux 下可用 `pthread_mutex_t` + `PTHREAD_PROCESS_SHARED` 属性实现跨进程互斥。

### 2.3 信号量

```cpp
// POSIX 信号量（线程/进程）
sem_t sem;
sem_init(&sem, 1, 1);   // pshared=1 表示进程间共享
sem_wait(&sem);          // P 操作
sem_post(&sem);          // V 操作

// 命名信号量（进程间）
sem_t* sem = sem_open("/mysem", O_CREAT, 0666, 1);
```

### 2.4 信号

```cpp
#include <signal.h>

// 信号处理函数注册（推荐 sigaction，更可控）
struct sigaction act = {};
act.sa_handler = handler;          // 或 sa_sigaction（可获取 siginfo_t）
sigaction(SIGTERM, &act, nullptr); // 优于 signal()，无歧义

// 发送信号
kill(pid, SIGUSR1);                // 给进程
pthread_kill(thread, SIGUSR1);     // 给线程

// 可靠的 async-signal-safe 函数列表
// 安全（可重入或原子）：write(), read(), _exit(), sigaction(), signal(),
//   memcpy(), strlen(), memset(), syscall()
// 不安全（不可重入）：malloc(), free(), printf(), fprintf(), mutex 操作
// **核心原则**：信号处理函数中只做 volatile sig_atomic_t 赋值 + write
```

### 2.5 事件通知 eventfd

```cpp
#include <sys/eventfd.h>

int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
// 写端：uint64_t val = 1; write(efd, &val, sizeof(val));
// 读端：uint64_t val; read(efd, &val, sizeof(val));
// 可用 epoll 监听 efd，实现事件驱动跨进程通知（比 pipe 更轻量，省去一对 fd）
```

### 2.6 Unix Domain Socket

```cpp
#include <sys/un.h>

int sock = socket(AF_UNIX, SOCK_STREAM, 0);  // SOCK_DGRAM 也可
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/mysocket");

bind(sock, (struct sockaddr*)&addr, sizeof(addr));
listen(sock, 5);
// 用法同 TCP socket，但不走网络协议栈，性能更高（同一主机）
// 支持 fd 传递（SCM_RIGHTS）：可跨进程传递文件描述符
```

### 2.7 文件锁 fcntl

```cpp
struct flock fl = {
    .l_type = F_WRLCK,    // F_RDLCK（读锁） / F_UNLCK（解锁）
    .l_whence = SEEK_SET,
    .l_start = 0,
    .l_len = 0,            // 0 表示锁整个文件
};
fcntl(fd, F_SETLK, &fl);   // 非阻塞加锁（返回 -1 表示冲突）
// fcntl(fd, F_SETLKW, &fl); // 阻塞加锁
// 进程退出时自动释放，适合跨进程互斥访问同一文件
```

### 2.8 跨进程同步原语速查

| 原语 | POSIX 接口 | 适用范围 | 说明 |
|:----:|:-----------|:--------:|:-----|
| **互斥锁** | `pthread_mutex_t` + `PTHREAD_PROCESS_SHARED` | 进程 | 放共享内存中，跨进程互斥 |
| **读写锁** | `pthread_rwlock_t` + `PROCESS_SHARED` | 进程 | 读多写少场景 |
| **信号量** | `sem_open` / `sem_init(pshared=1)` | 进程/线程 | P/V 操作，计数同步 |
| **条件变量** | `pthread_cond_t` + `PROCESS_SHARED` | 进程 | 需配 mutex 使用 |
| **原子操作** | `std::atomic` + C11 `atomic_*` | 进程/线程 | 需放共享内存，仅支持简单类型 |
| **文件锁** | `fcntl(fd, F_SETLK, &fl)` | 进程 | 进程退出自动释放 |
| **事件通知** | `eventfd` | 进程/线程 | 轻量，可用 epoll 监听 |
| **信号** | `sigaction` / `kill` | 进程 | 轻量但受限，信号处理函数不安全 |

---

## 三、守护进程（Daemon）创建

```cpp
#include <unistd.h>
#include <sys/stat.h>
#include <signal.h>

void daemonize() {
    pid_t pid = fork();
    if (pid > 0) exit(0);                  // ① 父进程退出，子进程成为孤儿
    setsid();                              // ② 创建新会话，脱离终端控制
    signal(SIGHUP, SIG_IGN);               // ③ 忽略 SIGHUP（会话 leader 退出时触发）
    fork();                                // ④ 第二次 fork，确保无法再获取终端

    chdir("/");                            // ⑤ 切换工作目录（释放挂载点）
    umask(0);                              // ⑥ 清除 umask，确保文件权限可控
    close(STDIN_FILENO);                   // ⑦ 关闭标准 fd
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    // 用 syslog 或文件记录日志
    openlog("mydaemon", LOG_PID, LOG_DAEMON);
    syslog(LOG_INFO, "daemon started");
}
```

**注意**：Linux 提供 `daemon(0, 0)` 库函数一键完成上述步骤，但面试中需手写流程。

---

## 四、Linux 系统调用分类速查

| 分类 | 主要系统调用 |
|------|-------------|
| **文件 I/O** | read、write、open、close、lseek、pread、pwrite、readv、writev |
| **I/O 多路复用** | poll、select、epoll_*、eventfd、timerfd、signalfd |
| **零拷贝** | sendfile、splice、tee、vmsplice、copy_file_range |
| **异步 I/O** | io_setup、io_submit、io_uring_* |
| **内存** | mmap、mprotect、munmap、brk、mlock、madvise |
| **网络** | socket、bind、listen、accept、connect、sendto、recvfrom |
| **IPC** | semget、shmget、msgget、mq_*、pipe |
| **定时器** | nanosleep、timer_create、timerfd_*、clock_* |
| **调度** | sched_*、futex、membarrier |
| **安全** | seccomp、getrandom、landlock、bpf |

---

## 五、IPC 快速决策表

| 场景 | 推荐方式 | 特点 |
|------|----------|------|
| 父子进程字节流 | 匿名管道（pipe） | 最简单 |
| 任意进程字节流 | 命名管道（FIFO） | 单向 |
| 任意进程双向流 | Unix Socket | 全双工 |
| 大量数据共享 | 共享内存（mmap） | 最快，需同步 |
| 进程同步 | 信号量（sem） | 计数同步 |
| 结构化消息 | POSIX 消息队列（mq_*） | 优先级 |
| 进程控制/通知 | 信号（signal） | 轻量但受限 |
| 跨主机 | 网络 Socket | 唯一选择 |

---

## 六、面试高频追问

### Q1: fork 后父子进程共享什么？写时复制如何工作？
- **共享**：代码段（只读）、文件描述符（指向同一偏移量）、mmap(MAP_SHARED)、信号处理器设置
- **私有（COW）**：数据段、栈、堆——fork 后共用物理页，标记为只读，任一进程写入时触发缺页异常，内核复制物理页后分别映射
- **优点**：fork 开销与进程大小无关（只复制页表），仅有写入的页才真正分配物理内存

### Q2: 写时复制（COW）的缺页流程？
1. fork 后父子进程的虚拟页映射到同一物理页，pte 标记为只读
2. 任一进程写入 → CPU 触发 page fault（保护错误）
3. 内核确认是 COW 场景 → 分配新物理页，复制原页内容
4. 更新该进程的页表，恢复读写权限
5. 返回用户态，重新执行写入指令

### Q3: 共享内存为什么需要同步？
- 多进程同时读写共享内存可能读到脏数据（写未完成时读）
- 必须配合：信号量、`pthread_mutex_t`（设 PROCESS_SHARED 属性）或原子操作
- 无锁编程可用 `std::atomic` + 内存序 + seqlock（适合读多写少场景）

### Q4: exit vs _exit 区别？为什么 fork 子进程必须用 _exit？
- `exit()`（C 库）：刷新 stdio 缓冲区 → 执行 atexit 回调 → 调用全局对象析构 → 内核 _exit
- `_exit()`（系统调用）：直接终止进程，不执行任何清理
- **子进程必须用 _exit**：父进程的 stdio 缓冲区可能还有未写入的数据，子进程 exit 会刷新这些缓冲区，导致父进程数据重复输出或混乱

### Q5: 信号安全的函数列表？信号处理函数有哪些限制？
- **安全**（async-signal-safe）：`write()`、`read()`、`_exit()`、`sigaction()`、`memcpy()`、`strlen()`、`syscall()`
- **不安全**：`malloc()`、`free()`、`printf()`、`fprintf()`、`mutex` 加解锁（可能死锁）
- **核心限制**：信号处理函数只能使用 async-signal-safe 函数，推荐做法是 `volatile sig_atomic_t flag = 1;` + 主循环检查

### Q6: 进程如何变成僵尸？如何避免？
- **产生条件**：子进程退出 → 内核保留进程表项（task_struct）→ 父进程未 wait → 永久僵尸
- **危害**：占进程表项（PID 有限），泄漏内核资源
- **防止僵尸**：
  - `signal(SIGCHLD, SIG_IGN)` — 内核自动回收（推荐，一句话解决）
  - 显式 `waitpid(-1, &status, WNOHANG)` 循环 — 需要知道子进程何时退出
  - 双重 fork — 孙进程由 init 回收（适用于不需要知道子进程退出状态的场景）

### Q7: 为什么 fork + exec 要分两步？有什么好处？
- **两步设计**：fork 复制当前进程 → (中间可做准备工作) → exec 替换为新的程序
- **中间可做的事情**：重定向 fd（dup2）、设置环境变量、切换用户（setuid）、设置资源限制（setrlimit）
- **典型应用**：
  - shell 管道 `|`：fork 后 dup2 重定向 stdout/stdin，再 exec 子命令
  - shell I/O 重定向 `>` `<`：fork 后 open/dup2 重定向 fd，再 exec
  - Nginx worker 热重启：fork 后保留 listen socket fd（设 FD_CLOEXEC），再 exec 新二进制

### Q8: daemon 创建为什么需要两次 fork？
- **第一次 fork**：父进程退出，子进程成为孤儿被 init 收养，脱离原终端
- **setsid()**：创建新会话，子进程成为新会话 leader，彻底脱离控制终端
- **第二次 fork**：确保子进程不是会话 leader（防止再次打开控制终端），此后进程不再与任何终端关联

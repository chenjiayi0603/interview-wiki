# 系统编程与 IPC

> POSIX API、Linux 系统调用、进程管理、进程间通信（管道/共享内存/信号量/信号/Socket）。

---

## 一、进程管理

### 1.1 fork + exec

```cpp
#include <unistd.h>
#include <sys/wait.h>

pid_t pid = fork();
if (pid < 0) { /* 错误 */ }
if (pid == 0) {
    // 子进程
    execl("/bin/echo", "echo", "hello", nullptr);
    _exit(1);
} else {
    // 父进程
    wait(nullptr);  // 等待子进程
}
```

- `fork()`：复制当前进程，子进程得到 0，父进程得到子进程 PID
- `exec` 族：替换当前进程映像（execl、execv、execvp、execle、execve）
- `wait()` / `waitpid()`：回收子进程
- `exit` vs `_exit`：exit 执行 atexit 回调+刷新流，_exit 直接内核终止

### 1.2 进程状态

```cpp
pid_t waitpid(pid, &status, options);
WIFEXITED(status)   // 正常退出
WEXITSTATUS(status) // 退出码
WIFSIGNALED(status) // 被信号终止
WTERMSIG(status)    // 终止信号
```

---

## 二、进程间通信（IPC）

### 2.1 管道

```cpp
// 匿名管道（父子进程）
int pipefd[2];
pipe(pipefd);  // pipefd[0] 读端，pipefd[1] 写端

// 命名管道（任意进程）
mkfifo("/tmp/myfifo", 0666);

// 进程池中用 pipe 分发任务
```

### 2.2 共享内存

```cpp
#include <sys/mman.h>

// POSIX 共享内存
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void* ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// 匿名共享内存（亲缘进程）
void* ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
```

**注意**：共享内存必须配合同步原语（mutex、信号量）使用。

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

// 信号处理
signal(SIGINT, handler);           // 简单注册
sigaction(SIGTERM, &act, nullptr); // 更可控

// 发送信号
kill(pid, SIGINT);          // 给进程
pthread_kill(thread, SIGUSR1);  // 给线程

// 可靠的异步信号安全函数列表（signal-safe）
// write(), read(), _exit(), sigaction()...
```

### 2.5 Unix Domain Socket

```cpp
#include <sys/un.h>

int sock = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/mysocket");
bind(sock, (struct sockaddr*)&addr, sizeof(addr));
listen(sock, 5);
```

---

## 三、Linux 系统调用分类速查

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

## 四、IPC 快速决策表

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

## 五、面试高频追问

### Q1: fork 后父子进程共享什么？
- 共享：代码段（只读）、文件描述符（指同一偏移量）、mmap（MAP_SHARED）
- 私有：数据段（COW）、栈、堆、信号处理器
- **写时复制（COW）**：fork 后物理内存共享，只有写入时才复制

### Q2: 共享内存为什么需要同步？
- 多进程同时读写共享内存可能读到脏数据
- 必须配合信号量、mutex（需设置 PTHREAD_PROCESS_SHARED 属性）或原子操作
- 无锁编程可用 `std::atomic` + 内存序 + seqlock

### Q3: 信号安全的函数列表？
- 安全：`write()`、`read()`、`_exit()`、`sigaction()`、`signal()`、`memcpy()`、`strlen()`
- 不安全：`malloc()`、`free()`、`printf()`、`mutex` 操作（可能死锁）
- **核心原则**：信号处理函数中只使用 async-signal-safe 函数

### Q4: 为什么 mmap 分配大内存比 malloc 快？
- malloc 大块（≥128KB）底层也是 mmap
- mmap 直接分配虚拟地址空间，物理内存延迟分配（缺页时真正占用）
- 释放时直接 munmap，不需要归还空闲链表

### Q5: exit vs _exit 区别？
- `exit()`：刷新 stdio 缓冲区、执行 atexit 注册函数、调用全局对象析构
- `_exit()`：直接内核终止进程，不执行任何清理
- 子进程 fork 后应使用 `_exit()` 避免误刷父进程缓冲区

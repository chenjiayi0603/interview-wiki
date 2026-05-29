# IPC 进程间通信

> 本文涵盖 Linux 下主流 IPC 方式：管道、消息队列、共享内存、信号量、套接字、文件锁与信号。

See also: [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]], [[POSIX进程控制]], [[POSIX信号处理]], [[POSIX网络编程]], [[C++多线程与并发]], [[POSIX线程管理]]

## 一、IPC 概述与选型

### 主流 IPC 方式总览

| 分类 | 主要方式 | 特点 |
|------|----------|------|
| 无名管道 | pipe | 父子/亲缘进程，半双工，字节流 |
| 命名管道 | FIFO | 任意进程，半双工 |
| 消息队列 | SysV / POSIX | 结构化消息，优先级，异步 |
| 共享内存 | SysV / POSIX | 最快，需同步 |
| 信号量 | SysV / POSIX | 同步/互斥 |
| 信号 | kill/sigaction | 事件/控制通知 |
| Socket | Unix Domain / 网络 | 全双工，跨机 |
| 文件锁 | fcntl/flock | 文件级互斥 |

### System V vs POSIX IPC

| 特性 | System V IPC | POSIX IPC |
|------|--------------|-----------|
| 命名 | key_t（ftok） | 文件路径 |
| API | shmget/semget/msgget | shm_open/sem_open/mq_open |
| 资源管理 | 手动清理（ipcrm） | 基于文件系统 |
| 推荐 | 兼容老系统 | ✅ 新项目优先 |

**结论**：新项目优先用 POSIX IPC；需兼容老系统或批量原子操作可用 System V。

### close 与 unlink

- **close**：关闭进程对 IPC 对象的引用，不删除对象
- **unlink / *_RMID**：删除系统级 IPC 对象，释放资源

### 选型建议

| 需求 | 推荐方式 |
|------|----------|
| 父子进程快速通信 | 匿名管道（pipe） |
| 任意进程本地通信 | FIFO 或 Unix Domain Socket |
| 大数据量/高性能 | 共享内存 + 信号量 |
| 结构化消息 | POSIX 消息队列 |
| 跨主机 | 网络 Socket |
| 文件资源控制 | 文件锁 |

[src: raw/ingested/2技术/cpp/C++多进程完整手册-三、IPC（进程间通信）完整指南.md]

---

## 二、管道

### 为什么管道是半双工？

- 本质是单向字节流缓冲区，写端写、读端读
- `pipe()` 返回两个 fd：一个只读、一个只写
- 需要双向通信时，建立两条管道

### 匿名管道（pipe）

```cpp
int pipefd[2];
pipe(pipefd);  // pipefd[0]读端, pipefd[1]写端

if (fork() == 0) {
    close(pipefd[1]);
    char buf[64];
    read(pipefd[0], buf, sizeof(buf) - 1);
    close(pipefd[0]);
} else {
    close(pipefd[0]);
    write(pipefd[1], "hello", 5);
    close(pipefd[1]);
    wait(nullptr);
}
```

- 仅限有亲缘关系的进程
- Linux 默认缓冲区约 64KB

### 命名管道（FIFO）

```cpp
mkfifo("myfifo", 0666);
int fd = open("myfifo", O_WRONLY);
write(fd, "data", 4);
close(fd);
// 另一进程：open("myfifo", O_RDONLY) 读取
unlink("myfifo");
```

- 任意进程可通信
- 以只读打开会阻塞直到有写端；以只写打开会阻塞直到有读端

### 匿名管道 vs 命名管道

| 对比项 | 匿名管道 | 命名管道 |
|--------|----------|----------|
| 作用范围 | 亲缘进程 | 任意进程 |
| 生命周期 | 进程结束即消失 | 文件存在即存在 |
| 性能 | 略高（纯内存） | 略低（涉及文件系统） |

[src: raw/ingested/2技术/cpp/C++多进程完整手册-三、IPC（进程间通信）完整指南.md]

---

## 三、消息队列

### System V 消息队列

**命名**：用 key（通常 `ftok` 生成）标识，任意进程可访问

```cpp
#include <sys/msg.h>
#include <sys/ipc.h>

struct msgbuf { long mtype; char mtext[32]; };
int mqid = msgget(1234, IPC_CREAT|0666);
struct msgbuf m = {.mtype=1};
strcpy(m.mtext, "msg");
msgsnd(mqid, &m, strlen(m.mtext)+1, 0);
msgrcv(mqid, &m, sizeof(m.mtext), 1, 0);
msgctl(mqid, IPC_RMID, NULL);
```

- 支持 `mtype` 消息类型
- 需用 `ftok` 生成 key

### POSIX 消息队列（推荐）

**命名**：路径式名字（如 `/mymq`），任意进程可 `mq_open` 访问

```cpp
#include <mqueue.h>

struct mq_attr attr = {.mq_maxmsg=10, .mq_msgsize=256};
mqd_t mq = mq_open("/mymq", O_CREAT|O_RDWR, 0666, &attr);
mq_send(mq, "hello", 5, 0);  // 第4参数为优先级
char buf[256];
mq_receive(mq, buf, sizeof(buf), NULL);
mq_close(mq);
mq_unlink("/mymq");
```

- 支持优先级、`mq_notify` 异步通知
- 编译需 `-lrt`

[src: raw/ingested/2技术/cpp/C++多进程完整手册-三、IPC（进程间通信）完整指南.md]

---

## 四、共享内存

### 本质

- 不同进程映射同一共享内存对象，虚拟地址可能不同，但指向同一物理内存
- **匿名**（`MAP_ANONYMOUS`）：仅亲缘进程，无文件
- **命名**（`shm_open`）：`/dev/shm/` 下文件，任意进程可访问

### POSIX 匿名共享内存 + 信号量

```cpp
struct Shared { int data; sem_t sem; };
Shared* shm = (Shared*)mmap(NULL, sizeof(Shared), PROT_READ|PROT_WRITE,
                            MAP_SHARED|MAP_ANONYMOUS, -1, 0);
sem_init(&shm->sem, 1, 0);  // 1=多进程共享

if (fork() == 0) {
    shm->data = 789;
    sem_post(&shm->sem);
    return 0;
}
sem_wait(&shm->sem);
printf("parent: data=%d\n", shm->data);
sem_destroy(&shm->sem);
munmap(shm, sizeof(Shared));
```

### POSIX 命名共享内存

```cpp
int fd = shm_open("/myshm", O_CREAT|O_RDWR, 0666);
ftruncate(fd, sizeof(SharedData));
SharedData* shared = (SharedData*)mmap(NULL, sizeof(SharedData),
    PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
// 使用 shared...
munmap(shared, sizeof(SharedData));
close(fd);
shm_unlink("/myshm");
```

### 共享内存 + pthread_spinlock（进程间）

```cpp
struct SharedData { int value; pthread_spinlock_t lock; };
// 共享内存中初始化
pthread_spin_init(&shared->lock, PTHREAD_PROCESS_SHARED);
// 使用后
pthread_spin_destroy(&shared->lock);
```

**注意**：spinlock 竞争激烈时耗 CPU，适合临界区极短场景。

### 共享内存环形队列（最简单示例）

```cpp
#include <sys/mman.h>
#include <pthread.h>
#include <cstring>
#include <cstdio>
#include <unistd.h>
#include <sys/wait.h>

static const int SLOTS = 8;
static const int SLOT_SIZE = 64;

struct ShmQueue {
    pthread_mutex_t mutex;
    int head, tail, count;
    char data[SLOTS][SLOT_SIZE];
};

bool push(ShmQueue* q, const char* msg) {
    pthread_mutex_lock(&q->mutex);
    if (q->count == SLOTS) { pthread_mutex_unlock(&q->mutex); return false; }
    strncpy(q->data[q->tail], msg, SLOT_SIZE - 1);
    q->tail = (q->tail + 1) % SLOTS;
    ++q->count;
    pthread_mutex_unlock(&q->mutex);
    return true;
}

bool pop(ShmQueue* q, char* out) {
    pthread_mutex_lock(&q->mutex);
    if (q->count == 0) { pthread_mutex_unlock(&q->mutex); return false; }
    strncpy(out, q->data[q->head], SLOT_SIZE);
    q->head = (q->head + 1) % SLOTS;
    --q->count;
    pthread_mutex_unlock(&q->mutex);
    return true;
}

int main() {
    ShmQueue* q = (ShmQueue*)mmap(NULL, sizeof(ShmQueue),
        PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);

    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);
    pthread_mutex_init(&q->mutex, &attr);
    pthread_mutexattr_destroy(&attr);
    q->head = q->tail = q->count = 0;

    if (fork() == 0) {
        push(q, "hello");
        push(q, "world");
        return 0;
    }
    wait(nullptr);
    char buf[SLOT_SIZE];
    while (pop(q, buf))
        printf("parent got: %s\n", buf);

    pthread_mutex_destroy(&q->mutex);
    munmap(q, sizeof(ShmQueue));
    return 0;
}
```

**要点：**
- `PTHREAD_PROCESS_SHARED`：mutex 必须设此属性才能跨进程使用
- 环形队列用 `head/tail/count` 管理，满则 push 返回 false
- 无流量费、无系统调用开销，是进程间最快的通信方式

[src: raw/ingested/2技术/cpp/C++多进程完整手册-三、IPC（进程间通信）完整指南.md]

---

## 五、信号量

### System V 信号量

```cpp
#include <sys/sem.h>
#include <sys/ipc.h>

// 创建信号量集
key_t key = ftok("/tmp/sem", 'a');
int semid = semget(key, 1, IPC_CREAT | 0666);

// 初始化
union semun {
    int val;
    struct semid_ds* buf;
    unsigned short* array;
} arg;
arg.val = 1;
semctl(semid, 0, SETVAL, arg);

// P操作
struct sembuf sb = {0, -1, 0};  // {信号量索引, 操作值, 标志}
semop(semid, &sb, 1);

// V操作
sb.sem_op = 1;
semop(semid, &sb, 1);

// 删除
semctl(semid, 0, IPC_RMID);
```

[src: raw/ingested/2技术/cpp/c++线程进程同步分析-进程同步机制.md]

### POSIX 命名信号量（进程间）

```cpp
#include <semaphore.h>
#include <fcntl.h>

// 创建或打开命名信号量
sem_t* sem = sem_open("/my_semaphore", O_CREAT, 0644, 1);

// 使用
sem_wait(sem);
// 临界区
sem_post(sem);

// 关闭
sem_close(sem);

// 删除（只有在所有进程都关闭后才真正删除）
sem_unlink("/my_semaphore");
```

[src: raw/ingested/2技术/cpp/c++线程进程同步分析-进程同步机制.md]

### POSIX 匿名信号量（进程间）

匿名信号量无名字，需放在**共享内存**中才能在进程间共享，`sem_init` 的 `pshared` 须为非 0。

匿名映射（`mmap`+`MAP_ANONYMOUS|MAP_SHARED`）可以实现进程间共享，但是**仅适用于有亲缘关系（如fork出来的子进程）**，因为映射的内核对象不会挂到文件系统，没有全局名，只有通过**继承文件描述符或映射**才能在多个进程间共享这块内存。

下面举例说明如何用匿名共享内存实现进程间信号量（无需全局名字，不用`shm_open`），适用于 fork 场景：

```cpp
#include <semaphore.h>
#include <sys/mman.h>
#include <unistd.h>
#include <iostream>

int main() {
    // 1. 匿名共享内存映射 —— 没有名字，不用shm_open
    sem_t* sem = (sem_t*)mmap(
        nullptr,                // 起始地址，通常用nullptr让内核决定
        sizeof(sem_t),          // 映射空间大小
        PROT_READ | PROT_WRITE, // 映射权限：可读可写
        MAP_SHARED | MAP_ANONYMOUS, // 共享+匿名映射
        -1,                     // 文件描述符（匿名映射要求为-1）
        0                       // 偏移量（通常为0）
    );

    // 2. 初始化信号量（只需一次）
    sem_init(sem, 1, 1);

    pid_t pid = fork();
    if (pid == 0) {
        // 子进程能继承这段匿名共享内存
        sem_wait(sem);
        std::cout << "Child in critical section" << std::endl;
        sleep(2);
        sem_post(sem);
        return 0;
    } else {
        sem_wait(sem);
        std::cout << "Parent in critical section" << std::endl;
        sleep(2);
        sem_post(sem);
        wait(nullptr);
    }

    // 3. 清理
    sem_destroy(sem);
    munmap(sem, sizeof(sem_t));
    return 0;
}
```

**说明：**
- 这种方式只有fork出的子进程才能访问和继承该匿名内存指针。如果两个无关进程需要同步，必须用名字（如shm_open/shmget等）。
- `MAP_ANONYMOUS`+`MAP_SHARED`的内存不会出现在文件系统，也不会保留任何名字，故为匿名共享内存。

```cpp
#include <semaphore.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <iostream>

int main() {
    // 1. 创建共享内存
    int fd = shm_open("/anon_sem_shm", O_CREAT | O_RDWR, 0666);
    ftruncate(fd, sizeof(sem_t));  // 必须设置大小，否则 SIGBUS

    // 2. 映射到进程地址空间
    sem_t* sem = (sem_t*)mmap(nullptr, sizeof(sem_t),
        PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

    // 3. 初始化：pshared=1 表示进程间共享，初值 1
    sem_init(sem, 1, 1);

    pid_t pid = fork();
    if (pid == 0) {
        // 子进程
        sem_wait(sem);
        std::cout << "Child: in critical section\n";
        sem_post(sem);
        return 0;
    }
    // 父进程
    sem_wait(sem);
    std::cout << "Parent: in critical section\n";
    sem_post(sem);
    wait(nullptr);

    // 4. 清理（仅父进程执行）
    sem_destroy(sem);
    munmap(sem, sizeof(sem_t));
    shm_unlink("/anon_sem_shm");
}
```
**要点**：匿名信号量必须位于 `mmap`/`shm_open` 的共享区域；`fork()` 后子进程继承映射，可访问同一 `sem_t`。

**结论：**
- 亲缘进程间（如fork/clone流派）可以用匿名共享内存+信号量。
- 完全独立进程之间则需通过带名字的共享内存（shm_open、shmget等）。

[src: raw/ingested/2技术/cpp/c++线程进程同步分析-进程同步机制.md]

### POSIX 未命名信号量（亲缘进程）

```cpp
sem_t sem;
sem_init(&sem, 1, 0);  // 1=多进程共享，初值0
if (fork() == 0) {
    sem_post(&sem);
    return 0;
}
sem_wait(&sem);
sem_destroy(&sem);
```

### POSIX 命名信号量（任意进程）

```cpp
sem_t* sem = sem_open("/mysem", O_CREAT, 0666, 0);
sem_post(sem);
sem_wait(sem);
sem_close(sem);
sem_unlink("/mysem");
```

### System V 信号量

```cpp
int semid = semget(3456, 1, IPC_CREAT|0666);
struct sembuf sb = {0, -1, 0};  // P操作
semop(semid, &sb, 1);
// 临界区
sb.sem_op = 1;  // V操作
semop(semid, &sb, 1);
semctl(semid, 0, IPC_RMID);
```

[src: raw/ingested/2技术/cpp/C++多进程完整手册-三、IPC（进程间通信）完整指南.md]

---

## 六、共享内存 + 互斥锁

### POSIX共享内存

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <pthread.h>

struct SharedData {
    pthread_mutex_t mutex;
    int counter;
};

// 进程1：创建共享内存
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, sizeof(SharedData));
SharedData* data = (SharedData*)mmap(nullptr, sizeof(SharedData), 
    PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// 初始化进程间互斥锁
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);
pthread_mutex_init(&data->mutex, &attr);
data->counter = 0;

// 进程2：打开共享内存
int fd = shm_open("/my_shm", O_RDWR, 0666);
SharedData* data = (SharedData*)mmap(nullptr, sizeof(SharedData), 
    PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// 使用
pthread_mutex_lock(&data->mutex);
data->counter++;
pthread_mutex_unlock(&data->mutex);

// 清理
munmap(data, sizeof(SharedData));
shm_unlink("/my_shm");
```

[src: raw/ingested/2技术/cpp/c++线程进程同步分析-进程同步机制.md]

---

## 七、文件锁

### fcntl锁

```cpp
#include <fcntl.h>
#include <unistd.h>

int fd = open("/tmp/lockfile", O_RDWR | O_CREAT, 0666);

// 加锁
struct flock fl;
fl.l_type = F_WRLCK;     // F_RDLCK读锁, F_WRLCK写锁, F_UNLCK解锁
fl.l_whence = SEEK_SET;
fl.l_start = 0;
fl.l_len = 0;            // 0表示锁定整个文件

// 阻塞加锁
fcntl(fd, F_SETLKW, &fl);

// 非阻塞加锁
int ret = fcntl(fd, F_SETLK, &fl);
if (ret == -1) {
    // 获取锁失败
}

// 解锁
fl.l_type = F_UNLCK;
fcntl(fd, F_SETLK, &fl);

close(fd);
```

### flock锁

```cpp
#include <sys/file.h>

int fd = open("/tmp/lockfile", O_RDWR | O_CREAT, 0666);

// 排他锁
flock(fd, LOCK_EX);

// 共享锁
flock(fd, LOCK_SH);

// 非阻塞
flock(fd, LOCK_EX | LOCK_NB);

// 解锁
flock(fd, LOCK_UN);

close(fd);
```

[src: raw/ingested/2技术/cpp/c++线程进程同步分析-进程同步机制.md]

---

## 八、套接字、文件锁与信号

### Unix Domain Socket（本地全双工）

```cpp
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>
#include <cstdio>
#include <cstring>

int main() {
    int sv[2];
    socketpair(AF_UNIX, SOCK_STREAM, 0, sv);
    if (fork() == 0) {
        close(sv[0]);
        write(sv[1], "hi", 2);
        close(sv[1]);
        return 0;
    }
    close(sv[1]);
    char buf[8] = {};
    ssize_t n = read(sv[0], buf, sizeof(buf) - 1);
    close(sv[0]);
    wait(nullptr);
    if (n > 0) printf("parent received: %s\n", buf);
    return 0;
}
```

### sendmsg/recvmsg 传递文件描述符

- 仅 **Unix 域流式套接字**（AF_UNIX + SOCK_STREAM）支持传递 fd；Unix Domain Socket 为**全双工**（同一 fd 可同时读写）
- TCP/IP 网络 socket **不能**传递 fd

```cpp
#define _GNU_SOURCE
#include <sys/socket.h>
#include <sys/un.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>
#include <cstdio>

void send_fd(int sock_fd, int fd_to_send) {
    struct msghdr msg = {};
    char cmsg_buf[CMSG_SPACE(sizeof(int))];
    msg.msg_control = cmsg_buf;
    msg.msg_controllen = sizeof(cmsg_buf);

    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_RIGHTS;
    cmsg->cmsg_len = CMSG_LEN(sizeof(int));
    *(int*)CMSG_DATA(cmsg) = fd_to_send;
    msg.msg_controllen = cmsg->cmsg_len;

    struct iovec iov = {.iov_base = (void*)"", .iov_len = 1};
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;

    sendmsg(sock_fd, &msg, 0);
}

int recv_fd(int sock_fd) {
    struct msghdr msg = {};
    char cmsg_buf[CMSG_SPACE(sizeof(int))];
    msg.msg_control = cmsg_buf;
    msg.msg_controllen = sizeof(cmsg_buf);

    char buf[1];
    struct iovec iov = {.iov_base = buf, .iov_len = 1};
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;

    recvmsg(sock_fd, &msg, 0);
    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg);
    if (cmsg && cmsg->cmsg_type == SCM_RIGHTS)
        return *(int*)CMSG_DATA(cmsg);
    return -1;
}

int main() {
    int sv[2];
    socketpair(AF_UNIX, SOCK_STREAM, 0, sv);
    if (fork() == 0) {
        close(sv[1]);
        int fd = recv_fd(sv[0]);
        write(fd, "child wrote\n", 11);
        close(fd);
        close(sv[0]);
        return 0;
    }
    close(sv[0]);
    int fd = open("/tmp/shared_file", O_CREAT|O_RDWR, 0666);
    send_fd(sv[1], fd);
    close(fd);
    close(sv[1]);
    wait(nullptr);
    return 0;
}
```

### 文件锁（fcntl / flock）

**flock 整个文件**（锁整个 fd）：

```cpp
#include <sys/file.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>
#include <cstdio>

int main() {
    int fd = open("/tmp/flock_ex", O_CREAT | O_RDWR | O_TRUNC, 0666);
    if (fd < 0) { perror("open"); return 1; }

    if (fork() == 0) {
        flock(fd, LOCK_EX);
        write(fd, "child\n", 6);
        flock(fd, LOCK_UN);
        close(fd);
        return 0;
    }

    flock(fd, LOCK_EX);
    write(fd, "parent\n", 7);
    flock(fd, LOCK_UN);

    wait(nullptr);
    close(fd);
    unlink("/tmp/flock_ex");
    return 0;
}
```

### 信号（sigaction 推荐）

```cpp
#include <signal.h>
#include <unistd.h>
#include <sys/wait.h>
#include <cstdio>

volatile sig_atomic_t g_done = 0;

void handler(int sig) {
    (void)sig;
    g_done = 1;
}

int main() {
    struct sigaction act = {};
    act.sa_handler = handler;
    sigemptyset(&act.sa_mask);
    sigaction(SIGUSR2, &act, NULL);

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }
    if (pid == 0) {
        sleep(1);
        kill(getppid(), SIGUSR2);
        return 0;
    }

    while (!g_done)
        pause();
    printf("parent: received SIGUSR2 from child %d\n", pid);
    wait(nullptr);
    return 0;
}
```

- 生产环境推荐 `sigaction` 替代 `signal`（`signal` 行为依赖实现，`sigaction` 可跨平台控制）
- 信号属于进程控制/事件通知，广义可算 IPC 的一种

[src: raw/ingested/2技术/cpp/C++多进程完整手册-三、IPC（进程间通信）完整指南.md]

## 九、快速决策表

| 类目       | 核心 API / 方式          | 用途                     |
| ---------- | ------------------------ | ------------------------ |
| 进程创建   | fork + exec, system, popen | 创建子进程、执行命令     |
| 进程等待   | wait, waitpid            | 回收子进程、获取退出状态 |
| 匿名管道   | pipe                     | 父子进程字节流、半双工   |
| 命名管道   | mkfifo                   | 任意进程半双工           |
| 消息队列   | msgget/msgsnd/msgrcv, mq_open | 结构化消息、优先级       |
| 共享内存（有锁） | shm_open/mmap + pthread_mutex(PROCESS_SHARED) | 多生产者/消费者，通用 |
| 共享内存（无锁） | mmap + std::atomic + placement new | SPSC，最高性能，无系统调用 |
| 信号量     | sem_init, sem_open, semget | 进程间同步/互斥          |
| Unix Socket | socketpair, AF_UNIX      | 本地全双工               |
| 文件锁     | fcntl, flock             | 文件级互斥               |
| 信号       | kill, sigaction          | 进程控制、事件通知       |
| 跨主机     | 网络 Socket (TCP/UDP)    | 跨进程、跨机器           |

[src: raw/ingested/2技术/cpp/C++多进程完整手册-八、快速决策表.md]

## Related Pages
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[POSIX进程控制]]
- [[POSIX信号处理]]
- [[POSIX网络编程]]
- [[C++多线程与并发]]
- [[POSIX线程管理]]
- [[C++POSIX文件操作]]

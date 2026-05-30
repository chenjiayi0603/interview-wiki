# POSIX 信号量

See also: [[C++多线程与并发]], [[C++POSIX文件操作]]

## 概述

POSIX 信号量提供进程间或线程间的同步机制，分为命名信号量和无名信号量两种。

## API 速查

### 命名信号量

```c
#include <semaphore.h>

sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value); // [POSIX]
int sem_close(sem_t *sem);                // [POSIX]
int sem_unlink(const char *name);         // [POSIX]
```

- `sem_open`：创建或打开一个命名信号量。`name` 通常以 `/` 开头（如 `/mysem`），`oflag` 可为 `O_CREAT`、`O_EXCL` 等，`mode` 为权限位，`value` 为初始值。
- `sem_close`：关闭信号量，释放进程内的资源，但不删除信号量。
- `sem_unlink`：从系统中删除命名信号量，当所有引用关闭后销毁。

### 无名信号量

```c
int sem_init(sem_t *sem, int pshared, unsigned int value); // [POSIX]
int sem_destroy(sem_t *sem);              // [POSIX]
```

- `sem_init`：初始化无名信号量。`pshared` 为 0 表示线程间共享，非 0 表示进程间共享（需放在共享内存中）。
- `sem_destroy`：销毁无名信号量。

### 操作

```c
int sem_wait(sem_t *sem);                 // [POSIX]   P 操作，阻塞等待
int sem_trywait(sem_t *sem);              // [POSIX]   非阻塞 P 操作
int sem_timedwait(sem_t *sem,
                  const struct timespec *abs_timeout); // [POSIX]  带超时的 P 操作
int sem_post(sem_t *sem);                 // [POSIX]   V 操作，增加信号量
int sem_getvalue(sem_t *sem, int *sval);  // [POSIX]   获取当前值
```

- `sem_wait`：若信号量值 > 0 则减 1 并返回；否则阻塞直到值 > 0。
- `sem_trywait`：非阻塞版本，若值 == 0 则立即返回 -1 并设 `errno = EAGAIN`。
- `sem_timedwait`：带绝对超时时间的等待，超时返回 -1 并设 `errno = ETIMEDOUT`。
- `sem_post`：将信号量值加 1，若有阻塞的 `sem_wait` 则唤醒其中一个。
- `sem_getvalue`：获取信号量当前值（注意：返回后值可能立即被其他线程改变）。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-10.-POSIX-IPC（新标准）.md]

## Related Pages
- [[C++多线程与并发]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[C++POSIX文件操作]]

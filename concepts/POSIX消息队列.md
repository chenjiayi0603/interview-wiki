# POSIX 消息队列

See also: [[C++多线程与并发]], [[POSIX信号量]], [[POSIX共享内存]]

## 概述

POSIX 消息队列提供进程间异步消息传递机制，支持带优先级的消息、超时发送/接收、以及异步通知。

## API 速查

### 创建与销毁

```c
#include <mqueue.h>

mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr); // [POSIX]
int mq_close(mqd_t mqdes);               // [POSIX]
int mq_unlink(const char *name);          // [POSIX]
```

- `mq_open`：创建或打开消息队列。`name` 以 `/` 开头，`oflag` 可为 `O_RDONLY`、`O_WRONLY`、`O_RDWR`、`O_CREAT`、`O_EXCL`、`O_NONBLOCK`。`attr` 指定队列属性（最大消息数、最大消息大小等），可为 NULL 使用默认值。
- `mq_close`：关闭消息队列描述符，不删除队列。
- `mq_unlink`：从系统中删除消息队列名称，当所有引用关闭后销毁。

### 发送与接收

```c
int mq_send(mqd_t mqdes, const char *msg_ptr,
            size_t msg_len, unsigned int msg_prio);    // [POSIX]
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr,
                   size_t msg_len, unsigned int *msg_prio); // [POSIX]
int mq_timedsend(mqd_t mqdes, const char *msg_ptr, size_t msg_len,
                 unsigned int msg_prio,
                 const struct timespec *abs_timeout);   // [POSIX]
ssize_t mq_timedreceive(mqd_t mqdes, char *msg_ptr, size_t msg_len,
                        unsigned int *msg_prio,
                        const struct timespec *abs_timeout); // [POSIX]
```

- `mq_send`：向队列发送消息，若队列满则阻塞（除非设置了 `O_NONBLOCK`）。`msg_prio` 指定优先级，数值越大优先级越高，高优先级消息优先投递。
- `mq_receive`：从队列接收消息，返回消息长度。若队列空则阻塞。`msg_prio` 返回接收消息的优先级。
- `mq_timedsend` / `mq_timedreceive`：带绝对超时时间的发送/接收。

### 属性与通知

```c
int mq_getattr(mqd_t mqdes, struct mq_attr *attr);     // [POSIX]
int mq_setattr(mqd_t mqdes, const struct mq_attr *newattr,
               struct mq_attr *oldattr);                // [POSIX]
int mq_notify(mqd_t mqdes, const struct sigevent *sevp); // [POSIX]
```

- `mq_getattr`：获取队列属性（`mq_flags`、`mq_maxmsg`、`mq_msgsize`、`mq_curmsgs`）。
- `mq_setattr`：设置队列属性（目前仅可设置 `O_NONBLOCK` 标志）。
- `mq_notify`：注册异步通知。当空队列收到消息时，通过指定方式（信号或线程）通知进程。一个队列同时只能有一个进程注册通知。

### `struct mq_attr` 结构

```c
struct mq_attr {
    long mq_flags;    // 0 或 O_NONBLOCK
    long mq_maxmsg;   // 最大消息数
    long mq_msgsize;  // 每条消息最大字节数
    long mq_curmsgs;  // 当前队列中消息数
};
```

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-10.-POSIX-IPC（新标准）.md]

## Related Pages
- [[C++多线程与并发]]
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[C++POSIX文件操作]]

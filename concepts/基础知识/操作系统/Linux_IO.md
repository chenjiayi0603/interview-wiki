# Linux IO

See also: [[内存管理]], [[性能优化]], [[C++网络编程]]

## 一、IO 方式对比

| IO方式 | 特点 | 适用场景 |
|--------|------|----------|
| `read/write` | 同步阻塞 | 简单IO |
| `pread/pwrite` | 原子定位读写 | 多线程文件操作 |
| `splice` | 零拷贝 | 管道转发 |
| `sendfile` | 零拷贝 | 文件到socket |
| `io_uring` | 异步IO | 高并发IO |

## 二、核心知识点

### 1. 传统 IO

```cpp
#include <unistd.h>
#include <fcntl.h>

// 同步阻塞读写
int fd = open("file.txt", O_RDONLY);
char buf[1024];
ssize_t n = read(fd, buf, sizeof(buf));
close(fd);

// 原子定位读写（线程安全）
ssize_t n = pread(fd, buf, sizeof(buf), offset);
ssize_t n = pwrite(fd, buf, sizeof(buf), offset);
```

### 2. 零拷贝技术

```cpp
#include <sys/sendfile.h>
#include <fcntl.h>

// sendfile：文件直接发送到 socket
int in_fd = open("file.txt", O_RDONLY);
int out_fd = socket(AF_INET, SOCK_STREAM, 0);
off_t offset = 0;
ssize_t sent = sendfile(out_fd, in_fd, &offset, file_size);

// splice：在两个 fd 之间移动数据
int pipefd[2];
pipe(pipefd);
// 从文件读到管道
splice(in_fd, NULL, pipefd[1], NULL, len, SPLICE_F_MOVE);
// 从管道写到 socket
splice(pipefd[0], NULL, out_fd, NULL, len, SPLICE_F_MOVE);
```

**零拷贝原理**：
- 传统方式：磁盘 → 内核缓冲区 → 用户缓冲区 → 内核 Socket 缓冲区 → 网卡
- sendfile：磁盘 → 内核缓冲区 → 内核 Socket 缓冲区 → 网卡（减少 2 次拷贝）
- splice：在内核空间直接移动数据，完全不经过用户态

### 3. io_uring（Linux 5.1+）

```cpp
#include <liburing.h>

// 初始化 io_uring
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

// 提交读请求
struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, size, offset);
io_uring_submit(&ring);

// 等待完成
struct io_uring_cqe* cqe;
io_uring_wait_cqe(&ring, &cqe);
int ret = cqe->res;  // 读取的字节数
io_uring_cqe_seen(&ring, cqe);

// 清理
io_uring_queue_exit(&ring);
```

**io_uring 优势**：
- 真正的异步 IO，不阻塞线程
- 批量提交和完成，减少系统调用
- 支持固定缓冲区和文件注册，减少内核开销

### 4. 阻塞 vs 非阻塞 vs 异步

| 模式 | 特点 | 示例 |
|------|------|------|
| 阻塞 IO | 等待数据就绪，线程挂起 | `read()` |
| 非阻塞 IO | 立即返回，需轮询 | `read()` + `O_NONBLOCK` |
| IO 多路复用 | 单线程监听多个 fd | `epoll` |
| 异步 IO | 内核通知完成 | `io_uring`、`aio` |

### 5. epoll 基础

```cpp
#include <sys/epoll.h>

int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;  // 可读事件
ev.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[MAX_EVENTS];
while (true) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < nfds; ++i) {
        if (events[i].data.fd == listen_fd) {
            // 接受新连接
        } else {
            // 处理数据
        }
    }
}
```

## 三、面试重点

1. **零拷贝**：减少用户态/内核态数据拷贝
2. **io_uring**：Linux 5.1+ 高性能异步IO
3. **阻塞 vs 非阻塞 vs 异步**

[src: raw/ingested/2技术/cpp/C++性能优化代码复习指南.md]

## Related Pages
- [[内存管理]]
- [[性能优化]]
- [[C++网络编程]]
- [[Boost.Asio]]

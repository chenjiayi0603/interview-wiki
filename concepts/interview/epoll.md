# epoll — 高性能 I/O 多路复用

> `[Linux 2.6+]`

See also: [[poll-select-pselect-ppoll]], [[eventfd]], [[timerfd]], [[signalfd]], [[C++网络编程]], [[Asio-epoll-与协程异步模型]]

## API 说明

```c
#include <sys/epoll.h>

int epoll_create(int size);            // [Linux 2.6+] size参数已被忽略（须>0）
int epoll_create1(int flags);          // [Linux 2.6.27+] flags: EPOLL_CLOEXEC
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // [Linux]
// op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);  // [Linux]
int epoll_pwait(int epfd, struct epoll_event *events, int maxevents,
                int timeout, const sigset_t *sigmask);   // [Linux 2.6.19+]
int epoll_pwait2(int epfd, struct epoll_event *events, int maxevents,
                 const struct timespec *timeout,
                 const sigset_t *sigmask);               // [Linux 5.11+]

struct epoll_event {
    uint32_t events;  // EPOLLIN, EPOLLOUT, EPOLLERR, EPOLLHUP, EPOLLET, EPOLLONESHOT, EPOLLRDHUP
    epoll_data_t data;
};
typedef union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

## 示例：epoll 服务器

```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>

#define MAX_EVENTS 64

void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

void epoll_server_example() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(9090)
    };
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, SOMAXCONN);
    set_nonblocking(server_fd);

    int epfd = epoll_create1(EPOLL_CLOEXEC);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = server_fd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

    struct epoll_event events[MAX_EVENTS];

    for (;;) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == server_fd) {
                int client_fd = accept(server_fd, NULL, NULL);
                if (client_fd >= 0) {
                    set_nonblocking(client_fd);
                    ev.events = EPOLLIN | EPOLLET; // 边缘触发
                    ev.data.fd = client_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
                }
            } else {
                char buf[1024];
                ssize_t n = read(events[i].data.fd, buf, sizeof(buf));
                if (n <= 0) {
                    epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                    close(events[i].data.fd);
                } else {
                    write(events[i].data.fd, buf, n); // echo
                }
            }
        }
    }
    close(epfd);
    close(server_fd);
}
```

## epoll 惊群问题及解决方案

**问题**：多进程/多线程在同一个 listen socket 上 `accept()` 时，新连接到来会唤醒所有阻塞的进程/线程，但只有其中一个能成功 accept，其余被无谓唤醒又失败返回，造成惊群。

**解决方案**：

| 方案 | 说明 | 内核版本 |
|------|------|----------|
| **SO_REUSEPORT** | 多进程绑定同一端口，内核按连接分配，只有被选中的进程收到事件 | Linux 3.9+ |
| **EPOLLEXCLUSIVE** | `epoll_ctl(EPOLL_CTL_ADD, EPOLLEXCLUSIVE)`，同一 fd 的多个 epoll 实例只唤醒其一 | Linux 4.5+ |
| **单进程多线程** | 单进程单 epoll，主线程 epoll_wait + 分发，无进程间 accept 惊群 | - |
| **accept 前加锁** | 传统做法：accept 前加 mutex，仅抢到锁者 accept，可减轻惊群 | - |

推荐使用 `SO_REUSEPORT` 或 `EPOLLEXCLUSIVE` 从内核层面避免 epoll 惊群。

[src: raw/ingested/2技术/cpp/Linux c系统调用-二、I-O-多路复用与事件驱动.md]

## Related Pages
- [[poll-select-pselect-ppoll]]
- [[eventfd]]
- [[timerfd]]
- [[signalfd]]
- [[C++网络编程]]
- [[Asio-epoll-与协程异步模型]]

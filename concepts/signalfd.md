# signalfd — 信号文件描述符

> `[Linux 2.6.22+]`

See also: [[epoll]], [[eventfd]], [[timerfd]], [[POSIX信号处理]]

## API 说明

```c
#include <sys/signalfd.h>

int signalfd(int fd, const sigset_t *mask, int flags); // [Linux 2.6.22+]
// fd=-1创建新的，>=0修改已有的
// flags: SFD_NONBLOCK | SFD_CLOEXEC
// read返回 struct signalfd_siginfo
```

## 示例

```c
#include <sys/signalfd.h>
#include <signal.h>
#include <unistd.h>
#include <stdio.h>

void signalfd_example() {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGINT);
    sigaddset(&mask, SIGTERM);
    sigprocmask(SIG_BLOCK, &mask, NULL); // 阻塞标准信号处理

    int sfd = signalfd(-1, &mask, SFD_CLOEXEC);

    printf("PID: %d, waiting for SIGINT or SIGTERM via signalfd...\n", getpid());

    struct signalfd_siginfo info;
    ssize_t n = read(sfd, &info, sizeof(info));
    if (n == sizeof(info)) {
        printf("Received signal %d from PID %d\n", info.ssi_signo, info.ssi_pid);
    }

    close(sfd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-二、I-O-多路复用与事件驱动.md]

## Related Pages
- [[epoll]]
- [[eventfd]]
- [[timerfd]]
- [[POSIX信号处理]]

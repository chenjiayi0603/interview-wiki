# timerfd — 定时器文件描述符

> `[Linux 2.6.25+]`

See also: [[epoll]], [[eventfd]], [[signalfd]], [[POSIX定时器与时钟]]

## API 说明

```c
#include <sys/timerfd.h>

int timerfd_create(int clockid, int flags);    // [Linux 2.6.25+]
// flags: TFD_NONBLOCK | TFD_CLOEXEC
int timerfd_settime(int fd, int flags,
                    const struct itimerspec *new_value,
                    struct itimerspec *old_value);  // [Linux 2.6.25+]
int timerfd_gettime(int fd, struct itimerspec *curr_value); // [Linux 2.6.25+]
```

## 示例：timerfd 配合 epoll

```c
#include <sys/timerfd.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <stdio.h>
#include <stdint.h>

void timerfd_example() {
    int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC);

    struct itimerspec its = {
        .it_value    = {1, 0},         // 首次1秒后
        .it_interval = {0, 500000000}  // 之后每500ms
    };
    timerfd_settime(tfd, 0, &its, NULL);

    for (int i = 0; i < 5; i++) {
        uint64_t expirations;
        read(tfd, &expirations, sizeof(expirations)); // 阻塞等待
        printf("Timer expired %lu time(s), count=%d\n", expirations, i + 1);
    }

    close(tfd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-二、I-O-多路复用与事件驱动.md]

## Related Pages
- [[epoll]]
- [[eventfd]]
- [[signalfd]]
- [[POSIX定时器与时钟]]

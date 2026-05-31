# eventfd — 事件通知

> `[Linux 2.6.22+]`

See also: [[epoll]], [[timerfd]], [[signalfd]], [[IPC进程间通信]]

## API 说明

```c
#include <sys/eventfd.h>

int eventfd(unsigned int initval, int flags);  // [Linux 2.6.22+]
// flags: EFD_CLOEXEC | EFD_NONBLOCK | EFD_SEMAPHORE
// 读写使用标准 read/write，操作uint64_t值
```

## 示例

```c
#include <sys/eventfd.h>
#include <unistd.h>
#include <stdio.h>
#include <stdint.h>
#include <pthread.h>

int efd;

void *eventfd_writer(void *arg) {
    for (int i = 1; i <= 3; i++) {
        uint64_t val = i;
        write(efd, &val, sizeof(val));
        printf("Writer: sent %lu\n", val);
        sleep(1);
    }
    return NULL;
}

void eventfd_example() {
    efd = eventfd(0, 0);

    pthread_t tid;
    pthread_create(&tid, NULL, eventfd_writer, NULL);

    for (int i = 0; i < 3; i++) {
        uint64_t val;
        read(efd, &val, sizeof(val)); // 阻塞等待
        printf("Reader: received %lu\n", val);
    }

    pthread_join(tid, NULL);
    close(efd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-二、I-O-多路复用与事件驱动.md]

## Related Pages
- [[epoll]]
- [[timerfd]]
- [[signalfd]]
- [[IPC进程间通信]]

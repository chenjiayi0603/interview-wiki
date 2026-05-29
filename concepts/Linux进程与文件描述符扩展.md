# Linux 进程与文件描述符扩展

> 本文涵盖 Linux 下进程与文件描述符扩展相关的系统调用 API：pidfd、memfd_create、close_range、memfd_secret。

See also: [[POSIX进程控制]], [[C++POSIX文件操作]], [[Linux内核与系统调用]], [[IPC进程间通信]]

---

## 一、pidfd — 进程文件描述符 `[Linux 5.1+]`

**API说明：**
```c
#include <sys/syscall.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>

int pidfd_open(pid_t pid, unsigned int flags);         // [Linux 5.3+]
int pidfd_send_signal(int pidfd, int sig,
                      siginfo_t *info, unsigned int flags); // [Linux 5.1+]
int pidfd_getfd(int pidfd, int targetfd,
                unsigned int flags);                   // [Linux 5.6+]

// 可配合 waitid/poll/epoll 使用
// waitid(P_PIDFD, pidfd, &siginfo, WEXITED);          // [Linux 5.4+]
```

**示例：**
```c
#define _GNU_SOURCE
#include <sys/syscall.h>
#include <unistd.h>
#include <signal.h>
#include <stdio.h>
#include <sys/wait.h>
#include <poll.h>

void pidfd_example() {
    pid_t pid = fork();
    if (pid == 0) {
        sleep(2);
        _exit(42);
    }

    int pidfd = syscall(SYS_pidfd_open, pid, 0);
    if (pidfd == -1) { perror("pidfd_open"); return; }

    // 使用poll等待进程退出
    struct pollfd pfd = { .fd = pidfd, .events = POLLIN };
    printf("Waiting for child via pidfd...\n");
    poll(&pfd, 1, -1);

    siginfo_t info;
    waitid(P_PIDFD, pidfd, &info, WEXITED);
    printf("Child exited with status: %d\n", info.si_status);

    close(pidfd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十六、进程与文件描述符扩展.md]

---

## 二、memfd_create — 匿名内存文件 `[Linux 3.17+]`

**API说明：**
```c
#include <sys/mman.h>

int memfd_create(const char *name, unsigned int flags); // [Linux 3.17+]
// flags: MFD_CLOEXEC | MFD_ALLOW_SEALING | MFD_HUGETLB

// 配合文件密封（File Sealing）                          // [Linux 3.17+]
#include <fcntl.h>
// fcntl(fd, F_ADD_SEALS, seals)
// seals: F_SEAL_SEAL | F_SEAL_SHRINK | F_SEAL_GROW | F_SEAL_WRITE
//        F_SEAL_FUTURE_WRITE [Linux 5.1+]
```

**示例：**
```c
#define _GNU_SOURCE
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>

void memfd_example() {
    int fd = memfd_create("my_memfd", MFD_CLOEXEC | MFD_ALLOW_SEALING);
    if (fd == -1) { perror("memfd_create"); return; }

    const char *data = "Hello from memfd!";
    write(fd, data, strlen(data));

    // 文件密封：禁止缩小和写入
    fcntl(fd, F_ADD_SEALS, F_SEAL_SHRINK | F_SEAL_GROW);

    char buf[64] = {0};
    lseek(fd, 0, SEEK_SET);
    read(fd, buf, sizeof(buf) - 1);
    printf("memfd content: %s\n", buf);

    // 可通过/proc/self/fd/%d或sendmsg传递给其他进程
    printf("memfd path: /proc/%d/fd/%d\n", getpid(), fd);

    close(fd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十六、进程与文件描述符扩展.md]

---

## 三、close_range — 批量关闭文件描述符 `[Linux 5.9+]`

**API说明：**
```c
#include <linux/close_range.h>

int close_range(unsigned int first, unsigned int last, unsigned int flags);
// flags: CLOSE_RANGE_UNSHARE | CLOSE_RANGE_CLOEXEC
```

**示例：**
```c
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>

void close_range_example() {
    // 关闭fd 3到1023（保留stdin/stdout/stderr）
    syscall(SYS_close_range, 3, 1023, 0);
    printf("Closed file descriptors 3-1023\n");
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十六、进程与文件描述符扩展.md]

---

## 四、memfd_secret — 秘密内存区域 `[Linux 5.14+]`

```c
#include <sys/mman.h>

int memfd_secret(unsigned int flags);  // 创建仅进程可访问的秘密内存
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十六、进程与文件描述符扩展.md]

## Related Pages
- [[POSIX进程控制]]
- [[C++POSIX文件操作]]
- [[Linux内核与系统调用]]
- [[IPC进程间通信]]
- [[内存管理]]
- [[userfaultfd]]
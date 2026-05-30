# Linux 文件系统与路径操作

> 本文涵盖 Linux 下文件系统与路径操作相关的系统调用 API：inotify、fanotify、*at 系列系统调用、statx。

See also: [[C++POSIX文件操作]], [[Linux系统调用-文件IO]], [[IPC进程间通信]]

---

## 一、inotify — 文件系统监控 `[Linux 2.6.13+]`

**API说明：**
```c
#include <sys/inotify.h>

int inotify_init(void);               // [Linux 2.6.13+]
int inotify_init1(int flags);         // [Linux 2.6.27+] flags: IN_NONBLOCK | IN_CLOEXEC
int inotify_add_watch(int fd, const char *pathname, uint32_t mask); // [Linux]
int inotify_rm_watch(int fd, int wd);  // [Linux]

// mask: IN_ACCESS | IN_MODIFY | IN_CREATE | IN_DELETE | IN_MOVED_FROM |
//       IN_MOVED_TO | IN_CLOSE_WRITE | IN_OPEN | IN_ATTRIB | IN_DELETE_SELF |
//       IN_MOVE_SELF | IN_ALL_EVENTS
```

**示例：监控目录变化**
```c
#include <sys/inotify.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

void inotify_example() {
    int ifd = inotify_init1(IN_CLOEXEC);
    int wd = inotify_add_watch(ifd, "/tmp",
        IN_CREATE | IN_DELETE | IN_MODIFY | IN_MOVED_FROM | IN_MOVED_TO);
    if (wd == -1) { perror("inotify_add_watch"); return; }

    printf("Monitoring /tmp for changes... (Ctrl+C to stop)\n");

    char buf[4096] __attribute__((aligned(__alignof__(struct inotify_event))));

    for (int count = 0; count < 10; count++) {
        ssize_t len = read(ifd, buf, sizeof(buf));
        if (len <= 0) break;

        for (char *ptr = buf; ptr < buf + len; ) {
            struct inotify_event *event = (struct inotify_event *)ptr;

            if (event->mask & IN_CREATE) printf("CREATE: ");
            if (event->mask & IN_DELETE) printf("DELETE: ");
            if (event->mask & IN_MODIFY) printf("MODIFY: ");
            if (event->mask & IN_MOVED_FROM) printf("MOVED_FROM: ");
            if (event->mask & IN_MOVED_TO) printf("MOVED_TO: ");

            if (event->len) printf("%s\n", event->name);

            ptr += sizeof(struct inotify_event) + event->len;
        }
    }

    inotify_rm_watch(ifd, wd);
    close(ifd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-三、文件系统与路径操作.md]

---

## 二、fanotify — 文件系统事件通知 `[Linux 2.6.37+]`

```c
#include <fcntl.h>
#include <sys/fanotify.h>

int fanotify_init(unsigned int flags, unsigned int event_f_flags);
int fanotify_mark(int fanotify_fd, unsigned int flags, uint64_t mask,
                  int dirfd, const char *pathname);
// 可监控文件访问、修改、打开等事件，常用于安全/审计
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-三、文件系统与路径操作.md]

---

## 三、*at 系列系统调用 (目录 fd 相对路径操作)

从 Linux 2.6.16 开始引入，解决了 TOCTTOU 竞态问题，支持以目录fd为基准使用相对路径。大部分 `*at` 函数已被纳入 POSIX.1-2008 标准。

**API说明：**
```c
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

int openat(int dirfd, const char *pathname, int flags, ...);  // [POSIX.1-2008]
int openat2(int dirfd, const char *pathname,
            struct open_how *how, size_t size);                // [Linux 5.6+]
int mkdirat(int dirfd, const char *pathname, mode_t mode);    // [POSIX.1-2008]
int unlinkat(int dirfd, const char *pathname, int flags);     // [POSIX.1-2008]
int renameat(int olddirfd, const char *oldpath,
             int newdirfd, const char *newpath);               // [POSIX.1-2008]
int renameat2(int olddirfd, const char *oldpath,
              int newdirfd, const char *newpath,
              unsigned int flags);                             // [Linux 3.15+]
int linkat(int olddirfd, const char *oldpath,
           int newdirfd, const char *newpath, int flags);      // [POSIX.1-2008]
int symlinkat(const char *target, int newdirfd,
              const char *linkpath);                           // [POSIX.1-2008]
ssize_t readlinkat(int dirfd, const char *pathname,
                   char *buf, size_t bufsiz);                  // [POSIX.1-2008]
int fchmodat(int dirfd, const char *pathname,
             mode_t mode, int flags);                          // [POSIX.1-2008]
int fchownat(int dirfd, const char *pathname,
             uid_t owner, gid_t group, int flags);             // [POSIX.1-2008]
int fstatat(int dirfd, const char *pathname,
            struct stat *statbuf, int flags);                  // [POSIX.1-2008]
int statx(int dirfd, const char *pathname, int flags,
          unsigned int mask, struct statx *statxbuf);          // [Linux 4.11+]
int faccessat(int dirfd, const char *pathname,
              int mode, int flags);                            // [POSIX.1-2008]
int faccessat2(int dirfd, const char *pathname,
               int mode, int flags);                           // [Linux 5.8+]
int utimensat(int dirfd, const char *pathname,
              const struct timespec times[2], int flags);      // [POSIX.1-2008]
// dirfd可以用AT_FDCWD表示当前工作目录                           // [POSIX.1-2008]
```

**示例：使用openat安全操作目录内文件**
```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/stat.h>

void openat_example() {
    int dirfd = open("/tmp", O_RDONLY | O_DIRECTORY);
    if (dirfd == -1) { perror("open dir"); return; }

    // 在 /tmp 下创建文件
    int fd = openat(dirfd, "at_test.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    write(fd, "hello openat\n", 13);
    close(fd);

    // 获取文件信息
    struct stat st;
    fstatat(dirfd, "at_test.txt", &st, 0);
    printf("File size: %ld\n", (long)st.st_size);

    // 删除
    unlinkat(dirfd, "at_test.txt", 0);
    close(dirfd);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-三、文件系统与路径操作.md]

---

## 四、statx — 增强版文件状态查询 `[Linux 4.11+]`

**API说明：**
```c
#include <sys/stat.h>
#include <linux/stat.h>

int statx(int dirfd, const char *pathname, int flags,
          unsigned int mask, struct statx *statxbuf);  // [Linux 4.11+]
// flags: AT_EMPTY_PATH | AT_SYMLINK_NOFOLLOW | AT_STATX_SYNC_AS_STAT | ...
// mask: STATX_TYPE | STATX_MODE | STATX_SIZE | STATX_ATIME | STATX_MTIME |
//       STATX_BTIME | STATX_ALL | ...  (STATX_BTIME = 文件创建时间，POSIX无此字段)
```

**示例：**
```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sys/stat.h>
#include <linux/stat.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <time.h>

void statx_example(const char *path) {
    struct statx stx;
    if (statx(AT_FDCWD, path, 0, STATX_ALL, &stx) == -1) {
        perror("statx");
        return;
    }

    printf("File: %s\n", path);
    printf("Size: %llu\n", (unsigned long long)stx.stx_size);
    printf("Blocks: %llu\n", (unsigned long long)stx.stx_blocks);
    printf("Links: %u\n", stx.stx_nlink);
    printf("UID: %u, GID: %u\n", stx.stx_uid, stx.stx_gid);

    if (stx.stx_mask & STATX_BTIME) {
        time_t t = stx.stx_btime.tv_sec;
        printf("Birth time: %s", ctime(&t));
    }
    time_t mt = stx.stx_mtime.tv_sec;
    printf("Modify time: %s", ctime(&mt));
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-三、文件系统与路径操作.md]

## Related Pages
- [[C++POSIX文件操作]]
- [[Linux系统调用-文件IO]]
- [[IPC进程间通信]]
- [[POSIX进程控制]]

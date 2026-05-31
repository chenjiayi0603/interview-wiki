# Linux 系统调用 - 文件与 I/O 操作

> 本文涵盖 Linux 下文件与 I/O 操作相关的系统调用 API，包括基础文件操作、文件描述符操作、文件控制与锁、文件元数据与属性、文件系统同步、扩展属性及文件建议与预读。

See also: [[C++POSIX文件操作]], [[IPC进程间通信]], [[C++多线程与并发]]

---

## 一、基础文件操作

```c
#include <unistd.h>
#include <fcntl.h>

ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
int open(const char *pathname, int flags, ...);  // mode_t mode
int close(int fd);
off_t lseek(int fd, off_t offset, int whence);   // SEEK_SET, SEEK_CUR, SEEK_END
int creat(const char *pathname, mode_t mode);    // 等价于 open(..., O_CREAT|O_TRUNC|O_WRONLY, mode)

// 分散/聚集 I/O
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
ssize_t pread(int fd, void *buf, size_t count, off_t offset);   // pread64
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);  // pwrite64
ssize_t preadv(int fd, const struct iovec *iov, int iovcnt, off_t offset);  // [2.6.30]
ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt, off_t offset);  // [2.6.30]
ssize_t preadv2(int fd, const struct iovec *iov, int iovcnt, off_t offset, int flags);  // [4.6]
ssize_t pwritev2(int fd, const struct iovec *iov, int iovcnt, off_t offset, int flags);  // [4.6]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

---

## 二、文件描述符操作

```c
#include <unistd.h>

int dup(int oldfd);
int dup2(int oldfd, int newfd);
int dup3(int oldfd, int newfd, int flags);       // [2.6.27] flags: O_CLOEXEC
int pipe(int pipefd[2]);
int pipe2(int pipefd[2], int flags);             // [2.6.27] flags: O_CLOEXEC | O_NONBLOCK
int close_range(unsigned int first, unsigned int last, unsigned int flags);  // [5.9+]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

---

## 三、文件控制与锁

```c
#include <fcntl.h>
#include <unistd.h>

int fcntl(int fd, int cmd, ...);
int flock(int fd, int operation);                // LOCK_SH, LOCK_EX, LOCK_UN, LOCK_NB
int fsync(int fd);
int fdatasync(int fd);
int truncate(const char *path, off_t length);
int ftruncate(int fd, off_t length);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

---

## 四、文件元数据与属性

```c
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int lchown(const char *pathname, uid_t owner, gid_t group);
mode_t umask(mode_t mask);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

---

## 五、文件系统同步

```c
#include <unistd.h>

void sync(void);
int syncfs(int fd);                              // [2.6.39]
int sync_file_range(int fd, off64_t offset, off64_t nbytes, unsigned int flags);  // [2.6.17]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

---

## 六、扩展属性 (xattr)

```c
#include <sys/xattr.h>

int setxattr(const char *path, const char *name, const void *value, size_t size, int flags);
int lsetxattr(const char *path, const char *name, const void *value, size_t size, int flags);
int fsetxattr(int fd, const char *name, const void *value, size_t size, int flags);
ssize_t getxattr(const char *path, const char *name, void *value, size_t size);
ssize_t lgetxattr(const char *path, const char *name, void *value, size_t size);
ssize_t fgetxattr(int fd, const char *name, void *value, size_t size);
ssize_t listxattr(const char *path, char *list, size_t size);
ssize_t llistxattr(const char *path, char *list, size_t size);
ssize_t flistxattr(int fd, char *list, size_t size);
int removexattr(const char *path, const char *name);
int lremovexattr(const char *path, const char *name);
int fremovexattr(int fd, const char *name);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

---

## 七、文件建议与预读

```c
#include <fcntl.h>

int posix_fadvise(int fd, off_t offset, off_t len, int advice);  // fadvise64
ssize_t readahead(int fd, off64_t offset, size_t count);          // [2.4.13]
int fallocate(int fd, int mode, off_t offset, off_t len);         // [2.6.23]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-一、文件与-I-O-操作.md]

## Related Pages
- [[C++POSIX文件操作]]
- [[IPC进程间通信]]
- [[C++多线程与并发]]
- [[POSIX进程控制]]

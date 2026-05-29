# Linux 系统调用 - 内存管理

> 本文涵盖 Linux 下内存管理相关的系统调用 API，包括虚拟内存操作、内存锁定、内存保护键、进程间内存访问等。

See also: [[内存管理]], [[IPC进程间通信]], [[C++多线程与并发]], [[POSIX共享内存]]

---

## 一、虚拟内存

```c
#include <sys/mman.h>
#include <unistd.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int mprotect(void *addr, size_t len, int prot);
int munmap(void *addr, size_t length);
int brk(void *addr);
void *mremap(void *old_address, size_t old_size, size_t new_size, int flags, ...);
int msync(void *addr, size_t length, int flags);
int mincore(void *addr, size_t length, unsigned char *vec);  // [2.4]
int madvise(void *addr, size_t length, int advice);          // [2.4]
int remap_file_pages(void *addr, size_t size, int prot,
                     size_t pgoff, int flags);    // [2.6] 已废弃
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-七、内存管理.md]

---

## 二、内存锁定

```c
#include <sys/mman.h>

int mlock(const void *addr, size_t len);
int munlock(const void *addr, size_t len);
int mlockall(int flags);                         // MCL_CURRENT | MCL_FUTURE
int munlockall(void);
int mlock2(const void *addr, size_t len, int flags);  // [4.4]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-七、内存管理.md]

---

## 三、内存保护键 (pkey) `[Linux 4.8+]`

```c
#include <sys/mman.h>

int pkey_alloc(unsigned int flags, unsigned int init_val);
int pkey_free(int pkey);
int pkey_mprotect(void *addr, size_t len, int prot, int pkey);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-七、内存管理.md]

---

## 四、进程间内存访问

```c
#include <sys/uio.h>

ssize_t process_vm_readv(pid_t pid, const struct iovec *local_iov,
                         unsigned long liovcnt, const struct iovec *remote_iov,
                         unsigned long riovcnt, unsigned long flags);   // [3.2]
ssize_t process_vm_writev(pid_t pid, const struct iovec *local_iov,
                          unsigned long liovcnt, const struct iovec *remote_iov,
                          unsigned long riovcnt, unsigned long flags);  // [3.2]
int process_madvise(int pidfd, const struct iovec *iovec, size_t vlen,
                   int advice, unsigned int flags);                     // [5.10]
int process_mrelease(int pidfd, unsigned int flags);                     // [5.15]
int mseal(void *addr, size_t len, unsigned int flags);                   // [6.10]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-七、内存管理.md]

## Related Pages
- [[内存管理]]
- [[IPC进程间通信]]
- [[C++多线程与并发]]
- [[POSIX共享内存]]
- [[Linux系统调用-文件IO]]
- [[进程内存区域与资源限制]]

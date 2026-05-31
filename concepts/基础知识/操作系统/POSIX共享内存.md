# POSIX 共享内存

See also: [[C++多线程与并发]], [[POSIX信号量]], [[内存管理]]

## 概述

POSIX 共享内存允许无关进程共享同一块物理内存区域，是最快的 IPC 方式。通常配合 `mmap` 和 `semaphore` 使用。

## API 速查

```c
#include <sys/mman.h>
#include <fcntl.h>

int shm_open(const char *name, int oflag, mode_t mode);  // [POSIX]
int shm_unlink(const char *name);                         // [POSIX]
```

- `shm_open`：创建或打开一个共享内存对象。`name` 通常以 `/` 开头，`oflag` 可为 `O_RDWR`、`O_CREAT`、`O_EXCL`、`O_TRUNC` 等，`mode` 为权限位。返回文件描述符。
- `shm_unlink`：删除共享内存对象名称，当所有引用关闭后销毁。

### 使用流程

1. `shm_open` 创建/打开共享内存对象，获得 fd。
2. `ftruncate(fd, size)` 设置共享内存大小。
3. `mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0)` 映射到进程地址空间。
4. 使用完毕后 `munmap` 解除映射，`close(fd)` 关闭文件描述符。
5. `shm_unlink` 删除共享内存对象。

### 与 System V 共享内存对比

| 特性 | POSIX `shm_open` | System V `shmget` |
|------|------------------|-------------------|
| 接口风格 | 文件描述符，类似文件操作 | 专用 key/id 体系 |
| 命名空间 | 文件系统路径（`/dev/shm`） | 整数 key |
| 与 mmap 配合 | 天然配合 | 需 `shmat` |
| 清理方式 | `shm_unlink` + `close` | `shmctl(IPC_RMID)` |

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-10.-POSIX-IPC（新标准）.md]

## Related Pages
- [[内存管理]]
- [[POSIX信号量]]
- [[POSIX消息队列]]
- [[C++POSIX文件操作]]

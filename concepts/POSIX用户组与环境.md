# POSIX 用户/组与环境

> 本文涵盖 POSIX 标准中用户/组管理、环境变量操作、系统信息查询相关的 C API。

See also: [[C++POSIX文件操作]], [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]], [[POSIX定时器与时钟]]

---

## 一、用户与组

### 1.1 用户/组 ID 获取与设置

```c
#include <unistd.h>
#include <sys/types.h>

uid_t getuid(void);                      // [POSIX] 获取实际用户 ID
uid_t geteuid(void);                     // [POSIX] 获取有效用户 ID
gid_t getgid(void);                      // [POSIX] 获取实际组 ID
gid_t getegid(void);                     // [POSIX] 获取有效组 ID
int setuid(uid_t uid);                   // [POSIX] 设置用户 ID
int seteuid(uid_t euid);                 // [POSIX] 设置有效用户 ID
int setgid(gid_t gid);                   // [POSIX] 设置组 ID
int setegid(gid_t egid);                 // [POSIX] 设置有效组 ID
int setreuid(uid_t ruid, uid_t euid);    // [POSIX XSI] 设置实际和有效用户 ID
int setregid(gid_t rgid, gid_t egid);    // [POSIX XSI] 设置实际和有效组 ID
int getgroups(int size, gid_t list[]);   // [POSIX] 获取附加组 ID 列表
```

**核心概念**：
- **实际 ID（real ID）**：进程所属的真实用户/组，通常为登录用户。
- **有效 ID（effective ID）**：用于权限检查的 ID，`setuid` 程序通过修改有效 ID 实现提权。
- `setreuid` / `setregid` 允许同时设置实际和有效 ID，常用于权限切换场景。

### 1.2 用户/组数据库查询

```c
#include <pwd.h>
#include <grp.h>

struct passwd *getpwnam(const char *name); // [POSIX] 按用户名查询
struct passwd *getpwuid(uid_t uid);        // [POSIX] 按 UID 查询
struct group *getgrnam(const char *name);  // [POSIX] 按组名查询
struct group *getgrgid(gid_t gid);         // [POSIX] 按 GID 查询
```

**注意**：这些函数返回指向静态内存的指针，**非线程安全**。多线程环境应使用 `getpwnam_r` / `getpwuid_r` 等可重入版本。

### 1.3 进程标识

```c
#include <unistd.h>

pid_t getpid(void);                        // [POSIX] 获取当前进程 ID
pid_t getppid(void);                       // [POSIX] 获取父进程 ID
pid_t getpgrp(void);                       // [POSIX] 获取进程组 ID
pid_t getsid(pid_t pid);                   // [POSIX XSI] 获取会话 ID
int setpgid(pid_t pid, pid_t pgid);        // [POSIX] 设置进程组 ID
pid_t setsid(void);                        // [POSIX] 创建新会话
```

**面试要点**：
- `setsid()` 用于创建守护进程：调用进程成为新会话的首进程和新进程组的组长，且脱离控制终端。
- 进程组用于作业控制（如 Shell 的前后台任务管理）。

---

## 二、环境变量

```c
#include <stdlib.h>

char *getenv(const char *name);             // [C/POSIX] 获取环境变量值
int setenv(const char *name, const char *value, int overwrite); // [POSIX] 设置环境变量
int unsetenv(const char *name);             // [POSIX] 删除环境变量
int putenv(char *string);                   // [POSIX] 添加/修改环境变量（"NAME=VALUE" 格式）
extern char **environ;                      // [POSIX] 全局环境变量指针数组
```

**关键区别**：
| 函数 | 行为 | 内存管理 |
|------|------|----------|
| `setenv` | 设置 `name=value`，若 `overwrite` 非 0 则覆盖已有值 | 内部分配内存拷贝 |
| `putenv` | 直接使用传入的字符串指针 | **不拷贝**，调用者需保证字符串生命周期 |
| `unsetenv` | 删除指定环境变量 | — |

**注意**：`putenv` 传入的字符串被环境直接引用，不能是栈上临时变量。`setenv` 更安全，推荐使用。

---

## 三、系统信息

### 3.1 系统配置查询

```c
#include <unistd.h>

long sysconf(long name);                   // [POSIX] 查询运行时系统限制/配置
```

常用 `name` 参数：
- `_SC_PAGESIZE`：内存页大小
- `_SC_NPROCESSORS_ONLN`：在线 CPU 数量
- `_SC_OPEN_MAX`：进程可打开的最大文件数

### 3.2 系统标识

```c
#include <sys/utsname.h>

int uname(struct utsname *buf);            // [POSIX] 获取系统信息
int gethostname(char *name, size_t len);   // [POSIX] 获取主机名
```

`struct utsname` 包含：`sysname`（OS 名）、`nodename`（主机名）、`release`（内核版本）、`version`、`machine`（硬件架构）。

### 3.3 资源限制

```c
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *rlim);  // [POSIX] 获取资源限制
int setrlimit(int resource, const struct rlimit *rlim); // [POSIX] 设置资源限制
int getrusage(int who, struct rusage *usage); // [POSIX XSI] 获取资源使用统计
```

`struct rlimit` 包含：
- `rlim_cur`：软限制（当前生效值）
- `rlim_max`：硬限制（软限制的上限）

常用 `resource` 类型：
- `RLIMIT_NOFILE`：最大打开文件数
- `RLIMIT_CORE`：core dump 文件大小上限
- `RLIMIT_DATA`：数据段大小上限
- `RLIMIT_STACK`：栈大小上限

**面试要点**：
- 普通进程只能调低硬限制、在硬限制范围内调整软限制；root 可提升硬限制。
- `getrusage` 可用于性能分析，获取进程的用户态/内核态 CPU 时间、缺页次数等。

---

## 四、面试高频问题

### Q1: 实际 UID 和有效 UID 的区别？
- **实际 UID**：标识进程的"真实所有者"，通常为启动进程的用户。
- **有效 UID**：用于权限检查（如文件访问），`setuid` 程序通过修改有效 UID 临时提权。
- 典型场景：`passwd` 命令（`setuid root`）以普通用户启动，但有效 UID 为 root，从而能修改 `/etc/shadow`。

### Q2: `setenv` 和 `putenv` 的区别？
- `setenv` 内部分配内存拷贝 `name` 和 `value`，调用者无需管理内存。
- `putenv` 直接使用传入的字符串指针，调用者必须保证字符串在环境变量使用期间有效（不能释放或覆盖）。
- **推荐使用 `setenv`**，避免悬挂指针问题。

### Q3: `setsid` 的作用？
- 创建新会话（session），调用进程成为会话首进程和进程组组长。
- 脱离控制终端，是**守护进程（daemon）化**的关键步骤。

### Q4: 如何获取系统的 CPU 核心数？
```c
long nprocs = sysconf(_SC_NPROCESSORS_ONLN);
```

### Q5: 如何设置进程可打开的最大文件数？
```c
struct rlimit rl;
rl.rlim_cur = 65536;
rl.rlim_max = 65536;
setrlimit(RLIMIT_NOFILE, &rl);
```

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-12.-用户-组与环境.md]

## Related Pages
- [[C++POSIX文件操作]]
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[POSIX定时器与时钟]]
- [[进程内存区域与资源限制]]
- [[Linux线程调度]]

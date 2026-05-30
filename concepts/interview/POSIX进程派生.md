# POSIX 进程派生 (posix_spawn)

> 本文涵盖 POSIX 标准中进程派生相关的 C API：`posix_spawn` / `posix_spawnp` 及配套的文件操作和属性设置函数。

See also: [[C++POSIX文件操作]], [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]], [[POSIX定时器与时钟]], [[POSIX用户组与环境]], [[POSIX终端IO]]

---

## 一、核心 API

```c
#include <spawn.h>

int posix_spawn(pid_t *pid, const char *path,
                const posix_spawn_file_actions_t *file_actions,
                const posix_spawnattr_t *attrp,
                char *const argv[], char *const envp[]);        // [POSIX]
int posix_spawnp(pid_t *pid, const char *file,
                 const posix_spawn_file_actions_t *file_actions,
                 const posix_spawnattr_t *attrp,
                 char *const argv[], char *const envp[]);       // [POSIX]
```

**`posix_spawn` vs `posix_spawnp`**：
- `posix_spawn`：`path` 参数为可执行文件的**绝对或相对路径**。
- `posix_spawnp`：`file` 参数为文件名，会在 `PATH` 环境变量中搜索可执行文件。

**参数说明**：
- `pid`：输出参数，返回子进程的 PID。
- `path` / `file`：可执行文件路径。
- `file_actions`：子进程的文件操作集合（可为 `NULL`）。
- `attrp`：子进程的属性（可为 `NULL`）。
- `argv`：参数列表（以 `NULL` 结尾）。
- `envp`：环境变量列表（以 `NULL` 结尾），若为 `NULL` 则继承父进程环境。

**与 `fork+exec` 的对比**：
- `posix_spawn` 将 `fork` 和 `exec` 合并为一个调用，在**某些系统上更高效**（避免 COW 开销）。
- 特别适合 `fork` 代价高昂的场景（如内存占用大的进程、无 MMU 的嵌入式系统）。

---

## 二、文件操作 (File Actions)

```c
int posix_spawn_file_actions_init(posix_spawn_file_actions_t *fa);      // [POSIX]
int posix_spawn_file_actions_destroy(posix_spawn_file_actions_t *fa);   // [POSIX]
int posix_spawn_file_actions_addopen(posix_spawn_file_actions_t *fa,
    int fildes, const char *path, int oflag, mode_t mode);              // [POSIX]
int posix_spawn_file_actions_addclose(posix_spawn_file_actions_t *fa,
    int fildes);                                                         // [POSIX]
int posix_spawn_file_actions_adddup2(posix_spawn_file_actions_t *fa,
    int fildes, int newfildes);                                          // [POSIX]
```

**文件操作类型**：

| 函数 | 作用 | 等效操作 |
|------|------|----------|
| `addopen` | 在子进程中打开文件 | `open(path, oflag, mode)` 并 dup 到 `fildes` |
| `addclose` | 在子进程中关闭文件描述符 | `close(fildes)` |
| `adddup2` | 在子进程中复制文件描述符 | `dup2(fildes, newfildes)` |

**典型用法**：重定向子进程的标准输入/输出/错误。

```c
posix_spawn_file_actions_t fa;
posix_spawn_file_actions_init(&fa);

// 打开输出文件并重定向 stdout
posix_spawn_file_actions_addopen(&fa, STDOUT_FILENO, "/tmp/out.log",
                                  O_WRONLY | O_CREAT | O_TRUNC, 0644);
// 关闭不需要的 fd
posix_spawn_file_actions_addclose(&fa, STDERR_FILENO);

posix_spawn(&pid, "/bin/ls", &fa, NULL, argv, NULL);
posix_spawn_file_actions_destroy(&fa);
```

---

## 三、进程属性 (Spawn Attributes)

```c
int posix_spawnattr_init(posix_spawnattr_t *attr);     // [POSIX]
int posix_spawnattr_destroy(posix_spawnattr_t *attr);  // [POSIX]
int posix_spawnattr_setflags(posix_spawnattr_t *attr, short flags); // [POSIX]
```

**常用 flags**：
- `POSIX_SPAWN_RESETIDS`：子进程的实际 UID/GID 重置为有效 UID/GID。
- `POSIX_SPAWN_SETPGROUP`：设置子进程的进程组。
- `POSIX_SPAWN_SETSCHEDPARAM` / `POSIX_SPAWN_SETSCHEDULER`：设置调度策略/参数。
- `POSIX_SPAWN_SETSIGDEF` / `POSIX_SPAWN_SETSIGMASK`：信号处理相关。

**其他属性设置函数**（POSIX 扩展）：
```c
int posix_spawnattr_setpgroup(posix_spawnattr_t *attr, pid_t pgroup);
int posix_spawnattr_setschedparam(posix_spawnattr_t *attr,
                                   const struct sched_param *schedparam);
int posix_spawnattr_setschedpolicy(posix_spawnattr_t *attr, int policy);
int posix_spawnattr_setsigdefault(posix_spawnattr_t *attr,
                                   const sigset_t *sigdefault);
int posix_spawnattr_setsigmask(posix_spawnattr_t *attr,
                                const sigset_t *sigmask);
```

---

## 四、面试高频问题

### Q1: `posix_spawn` 和 `fork+exec` 的区别？
- `fork+exec`：两步操作，`fork` 复制整个进程空间（COW 优化），`exec` 替换。
- `posix_spawn`：一步完成，在某些系统（尤其是无 MMU 的嵌入式系统）上更高效，避免不必要的内存复制。
- **选择建议**：内存占用大的进程、嵌入式系统优先考虑 `posix_spawn`；需要 `fork` 后 `exec` 前做复杂操作的场景用 `fork+exec`。

### Q2: `posix_spawn` 和 `posix_spawnp` 的区别？
- `posix_spawn`：使用绝对或相对路径。
- `posix_spawnp`：在 `PATH` 中搜索可执行文件，类似 `execvp`。

### Q3: 如何用 `posix_spawn` 重定向子进程的 stdout？
使用 `posix_spawn_file_actions_addopen` 将 `STDOUT_FILENO` 映射到目标文件。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-17.-进程派生-(posix_spawn).md]

## Related Pages
- [[C++POSIX文件操作]]
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[POSIX定时器与时钟]]
- [[POSIX用户组与环境]]
- [[POSIX终端IO]]
- [[Linux线程调度]]

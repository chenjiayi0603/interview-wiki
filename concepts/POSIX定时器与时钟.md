# POSIX 定时器与时钟

See also: [[C++POSIX文件操作]], [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]], [[C++多线程与并发]]

## 一、POSIX 时钟

```c
#include <time.h>

int clock_gettime(clockid_t clk_id, struct timespec *tp);          // [POSIX]
int clock_settime(clockid_t clk_id, const struct timespec *tp);    // [POSIX]
int clock_getres(clockid_t clk_id, struct timespec *res);          // [POSIX]
// clk_id: CLOCK_REALTIME, CLOCK_MONOTONIC, CLOCK_PROCESS_CPUTIME_ID, CLOCK_THREAD_CPUTIME_ID
```

### 时钟类型 (clk_id)

| 时钟 ID | 说明 |
|---------|------|
| `CLOCK_REALTIME` | 系统实时时钟，可被用户或 NTP 修改 |
| `CLOCK_MONOTONIC` | 单调时钟，不受系统时间调整影响，适合测量时间间隔 |
| `CLOCK_PROCESS_CPUTIME_ID` | 进程 CPU 时间 |
| `CLOCK_THREAD_CPUTIME_ID` | 线程 CPU 时间 |

### API 说明

- **`clock_gettime`**：获取指定时钟的当前时间，存入 `timespec` 结构（秒 + 纳秒）。
- **`clock_settime`**：设置指定时钟的时间（需要相应权限，`CLOCK_MONOTONIC` 不可设置）。
- **`clock_getres`**：获取指定时钟的精度（分辨率）。

---

## 二、POSIX 定时器

```c
#include <signal.h>
#include <time.h>

int timer_create(clockid_t clockid, struct sigevent *sevp, timer_t *timerid); // [POSIX]
int timer_settime(timer_t timerid, int flags,
                  const struct itimerspec *new_value,
                  struct itimerspec *old_value);   // [POSIX]
int timer_gettime(timer_t timerid, struct itimerspec *curr_value); // [POSIX]
int timer_getoverrun(timer_t timerid);             // [POSIX]
int timer_delete(timer_t timerid);                 // [POSIX]

// 传统定时器（已标记过时）
int setitimer(int which, const struct itimerval *new_value,
              struct itimerval *old_value);         // [POSIX XSI]
int getitimer(int which, struct itimerval *curr_value); // [POSIX XSI]
```

### 核心 API

- **`timer_create`**：创建一个定时器。
  - `clockid`：指定使用的时钟（如 `CLOCK_REALTIME`、`CLOCK_MONOTONIC`）。
  - `sevp`：指定定时器到期时的通知方式（信号、线程等）。
  - `timerid`：返回定时器 ID。
- **`timer_settime`**：启动或修改定时器。
  - `flags`：`0` 为相对时间，`TIMER_ABSTIME` 为绝对时间。
  - `new_value`：`it_interval`（周期）和 `it_value`（首次到期时间）。
- **`timer_gettime`**：获取定时器剩余时间。
- **`timer_getoverrun`**：获取定时器超限次数（上次信号未处理完又到期）。
- **`timer_delete`**：删除定时器并释放资源。

### 传统定时器（已过时）

- **`setitimer` / `getitimer`**：POSIX XSI 扩展，基于 `SIGALRM`/`SIGVTALRM`/`SIGPROF` 信号。
- 功能有限（每个进程只能有 3 个定时器），推荐使用 `timer_create` 系列。

---

## 三、睡眠函数

```c
#include <unistd.h>
#include <time.h>

unsigned int sleep(unsigned int seconds);    // [POSIX]
int usleep(useconds_t usec);                // [POSIX] 已在POSIX.1-2008中移除，推荐nanosleep
int nanosleep(const struct timespec *req, struct timespec *rem);   // [POSIX]
int clock_nanosleep(clockid_t clock_id, int flags,
                    const struct timespec *request,
                    struct timespec *remain);                       // [POSIX]
```

### API 对比

| 函数 | 精度 | 可中断 | 说明 |
|------|------|--------|------|
| `sleep` | 秒级 | 是（返回剩余秒数） | 最基础，可能基于 `SIGALRM` 实现 |
| `usleep` | 微秒级 | 是 | **已在 POSIX.1-2008 中移除**，不推荐使用 |
| `nanosleep` | 纳秒级 | 是（`rem` 返回剩余时间） | 推荐替代 `usleep`，不依赖信号 |
| `clock_nanosleep` | 纳秒级 | 是 | 可指定时钟类型（如 `CLOCK_MONOTONIC`），支持绝对/相对时间 |

### 使用建议

- **高精度睡眠**：使用 `clock_nanosleep`，指定 `CLOCK_MONOTONIC` 避免系统时间调整影响。
- **可移植代码**：使用 `nanosleep` 替代已废弃的 `usleep`。
- **绝对时间唤醒**：`clock_nanosleep` 配合 `TIMER_ABSTIME` 可精确在某个时间点唤醒，避免相对时间累积误差。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-11.-定时器与时钟.md]

## Related Pages
- [[C++POSIX文件操作]]
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[C++多线程与并发]]
- [[Linux线程调度]]

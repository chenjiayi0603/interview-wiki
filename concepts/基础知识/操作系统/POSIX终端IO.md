# POSIX 终端 I/O (termios)

> 本文涵盖 POSIX 标准中终端 I/O 相关的 C API，包括终端属性控制、波特率设置、行控制、终端标识与前台进程组管理。

See also: [[C++POSIX文件操作]], [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]], [[POSIX定时器与时钟]], [[POSIX用户组与环境]]

---

## 一、终端属性获取与设置

```c
#include <termios.h>
#include <unistd.h>

int tcgetattr(int fd, struct termios *termios_p);    // [POSIX] 获取终端属性
int tcsetattr(int fd, int optional_actions,
              const struct termios *termios_p);       // [POSIX] 设置终端属性
```

**核心概念**：
- `tcgetattr` 获取与文件描述符 `fd` 关联的终端当前属性，存入 `termios` 结构体。
- `tcsetattr` 根据 `optional_actions` 参数设置终端属性：
  - `TCSANOW`：立即生效。
  - `TCSADRAIN`：等待所有输出传输完毕后再生效。
  - `TCSAFLUSH`：等待输出传输完毕，并丢弃所有未读取的输入数据。

**面试要点**：
- `termios` 结构体包含 `c_iflag`（输入标志）、`c_oflag`（输出标志）、`c_cflag`（控制标志）、`c_lflag`（本地标志）和 `c_cc`（控制字符数组）。
- 典型用途：配置终端为原始模式（raw mode）或规范模式（canonical mode），实现自定义终端交互程序。

---

## 二、波特率控制

```c
speed_t cfgetispeed(const struct termios *termios_p); // [POSIX] 获取输入波特率
speed_t cfgetospeed(const struct termios *termios_p); // [POSIX] 获取输出波特率
int cfsetispeed(struct termios *termios_p, speed_t speed); // [POSIX] 设置输入波特率
int cfsetospeed(struct termios *termios_p, speed_t speed); // [POSIX] 设置输出波特率
```

**核心概念**：
- `speed_t` 是波特率类型，常用值包括 `B9600`、`B115200` 等。
- 输入和输出波特率可以独立设置，但大多数场景下两者相同。

**面试要点**：
- 波特率（baud rate）是串行通信中每秒传输的符号数，与比特率（bps）在常见调制方式下等价。
- 设置波特率前通常先调用 `tcgetattr` 获取当前配置，修改后再调用 `tcsetattr` 使其生效。

---

## 三、行控制操作

```c
int tcdrain(int fd);       // [POSIX] 等待所有输出传输完毕
int tcflush(int fd, int queue_selector); // [POSIX] 丢弃指定队列中的数据
int tcflow(int fd, int action);          // [POSIX] 控制数据流
int tcsendbreak(int fd, int duration);   // [POSIX] 发送 BREAK 信号
```

**核心概念**：
- `tcdrain`：阻塞直到所有已写入的输出数据被实际传输完毕，常用于确保数据完整发送后再关闭连接。
- `tcflush`：根据 `queue_selector` 丢弃缓冲区数据：
  - `TCIFLUSH`：丢弃已接收但未读取的输入数据。
  - `TCOFLUSH`：丢弃已写入但未传输的输出数据。
  - `TCIOFLUSH`：同时丢弃输入和输出缓冲区数据。
- `tcflow`：控制数据流方向：
  - `TCOOFF`：暂停输出。
  - `TCOON`：恢复输出。
  - `TCIOFF`：发送 STOP 字符，使对方暂停发送。
  - `TCION`：发送 START 字符，使对方恢复发送。
- `tcsendbreak`：发送持续时间为 `duration`（通常为 0）的 BREAK 信号，用于串行通信中的特殊信令。

**面试要点**：
- `tcdrain` 与 `tcflush` 的区别：`tcdrain` 是等待数据发送完毕，`tcflush` 是主动丢弃缓冲区数据。
- BREAK 信号是串行通信中的特殊状态（线路保持逻辑 0 超过一个字符帧的时间），用于唤醒对方或作为带外信令。

---

## 四、终端标识与前台进程组

```c
int isatty(int fd);        // [POSIX] 判断文件描述符是否关联终端
char *ttyname(int fd);     // [POSIX] 获取终端设备名称
pid_t tcgetpgrp(int fd);   // [POSIX] 获取终端的前台进程组 ID
int tcsetpgrp(int fd, pid_t pgrp); // [POSIX] 设置终端的前台进程组 ID
```

**核心概念**：
- `isatty`：判断 `fd` 是否指向一个终端设备，常用于检测程序是否在交互式终端中运行（如区分管道输入和终端输入）。
- `ttyname`：返回终端设备文件名（如 `/dev/pts/0`），返回的指针指向静态内存，非线程安全。
- `tcgetpgrp` / `tcsetpgrp`：用于作业控制（job control），Shell 通过它们管理前台进程组，决定哪个进程组可以接收终端输入和信号（如 SIGINT、SIGTSTP）。

**面试要点**：
- `isatty` 的典型应用：程序根据是否在终端中运行来调整输出格式（如是否输出颜色代码）。
- 前台进程组：终端同一时刻只有一个前台进程组，该组内的进程可以读取终端输入；后台进程组尝试读取终端时会收到 SIGTTIN 信号。
- `tcsetpgrp` 通常由 Shell 在作业控制中调用，普通进程较少直接使用。

---

## 五、面试高频问题

### Q1: `tcgetattr` 和 `tcsetattr` 的作用？
- `tcgetattr` 获取终端当前配置（如输入/输出标志、控制字符等）。
- `tcsetattr` 设置终端配置，通过 `optional_actions` 控制生效时机（立即/等待输出完成/同时清空输入）。
- 典型场景：将终端从规范模式切换为原始模式，实现逐字符读取（如 vim、less 等程序）。

### Q2: 什么是终端原始模式（raw mode）？
- 原始模式下，终端不进行行缓冲、不回显、不处理特殊字符（如 Ctrl+C 不产生 SIGINT），输入数据逐字符直接传递给应用程序。
- 通过修改 `termios` 结构体的 `c_lflag` 清除 `ICANON`、`ECHO`、`ISIG` 等标志实现。

### Q3: `tcdrain` 和 `tcflush` 的区别？
- `tcdrain`：阻塞等待所有输出数据发送完毕，不丢弃数据。
- `tcflush`：主动丢弃输入/输出缓冲区中的数据，不等待。

### Q4: `isatty` 的用途？
- 判断文件描述符是否关联终端设备。
- 典型用途：程序检测是否在交互式终端中运行，以决定是否输出彩色日志、进度条等。

### Q5: 前台进程组和后台进程组的区别？
- 前台进程组：可以读取终端输入，接收终端产生的信号（如 Ctrl+C 产生 SIGINT）。
- 后台进程组：尝试读取终端时会收到 SIGTTIN 信号（默认暂停进程）。
- Shell 通过 `tcsetpgrp` 切换前台进程组实现作业控制。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-13.-终端-I-O-(termios).md]

## Related Pages
- [[C++POSIX文件操作]]
- [[POSIX信号量]]
- [[POSIX共享内存]]
- [[POSIX消息队列]]
- [[POSIX定时器与时钟]]
- [[POSIX用户组与环境]]
- [[Linux线程调度]]

# C标准库实现

See also: [[内存管理]], [[C++语言特性]], [[C++进阶知识点]]

## 概述

C 标准库本质是一组 **目标文件 + 汇编优化文件 + 启动代码（crt*.o）+ 头文件** 的集合，最终打包为静态库 `libc.a` 和/或共享库 `libc.so`。

- **与系统调用的关系**：标准库函数大多是**对系统调用的封装 + 缓冲 + 额外语义**，大量逻辑完全在用户态实现，只有在必须与内核交互时才发起系统调用。
- **链接流程**（Linux + gcc + glibc）：链接器自动链接 `crt1.o`、`crti.o`、`crtn.o` 等启动文件，程序入口是库提供的 `_start`，完成初始化后调用 `main`。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 内存管理：malloc/free/calloc/realloc

### malloc

```c
void *malloc(size_t size);
```

**核心实现思路：**
- 维护一套**堆管理器**，把从内核申请来的大块内存（通常通过 `brk` 或 `mmap`）切成许多小块（chunk）；
- 使用多种 **空闲链表 / bins** 存放不同大小的空闲块（fast bins、small bins、large bins 等）；
- `malloc` 调用时：优先在空闲块里寻找合适的块（best-fit 或近似策略），若找到则标记为已用，必要时分裂出剩余空闲块；若找不到则向内核申请新内存区域。

**与系统调用的关系：**
- 如果空闲块足够，`malloc` 完全在**用户态**完成，不发生系统调用；
- 只有当堆空间不足时，才调用 `brk`/`sbrk`（调整进程数据段）或 `mmap(MAP_ANONYMOUS)`（映射匿名内存，特别是大块分配）。

### free

```c
void free(void *ptr);
```

- 根据指针找到对应 chunk 的**头部元数据**；
- 将其标记为空闲，并尝试与前后相邻空闲块合并（减少碎片）；
- 将合并后的空闲块挂入相应大小类别的空闲链表中；
- 若空闲块位于堆顶或是 `mmap` 得到的独立区域且足够大，可能调用 `brk` 收缩堆顶或 `munmap` 归还给内核。
- 并不是一 `free` 就必然进行系统调用，绝大多数释放只在用户态完成。

### calloc

```c
void *calloc(size_t nmemb, size_t size);
```

- 计算总大小 `total = nmemb * size`，需检查溢出；
- 调用 `malloc(total)`；
- 若成功，将返回区域全部清零（小块可用 `memset`，若底层从内核以 `mmap` 拿到的物理页默认已清零，可部分省略）。

### realloc

```c
void *realloc(void *ptr, size_t new_size);
```

- 若 `ptr == NULL`，等价于 `malloc(new_size)`；
- 若 `new_size == 0`，等价于 `free(ptr)`；
- 否则：查看当前块能否就地扩展（后面紧邻块空闲且可合并），如果可以则扩展并更新元数据；如果不行则 `malloc` 新块 → `memcpy` 拷贝旧数据 → `free` 旧块。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 字符串与内存操作：str* / mem*

这一组函数大多不需要系统调用，而是**纯用户态 + 汇编优化**。

### strlen / strcpy / strncpy / strcat / strcmp

- **`strlen`**：从指针开始逐字节扫描直到 `'\0'`，实际实现常用按字/按机器字长读取 + 位运算快速检测 0 字节，并使用 CPU 指令特性（如 `REPNE SCASB`、`SSE/AVX`）。
- **`strcpy` / `strcat`**：`strcpy(dst, src)` 拷贝 `src` 直到包括结尾 `'\0'`；`strcat(dst, src)` 先 `strlen(dst)` 找尾部再复制 `src`。
- **`strcmp`**：逐字节比较直到字符不同或遇到 `'\0'`，返回 `(unsigned char)s1[i] - (unsigned char)s2[i]`，高级实现用按字比较 + 按位检测差异。

### mem* 系列：memcpy / memmove / memset / memcmp

- **`memcpy`**：假设源和目的区域**不重叠**，先对齐到机器字边界，再按机器字/向量宽度批量拷贝，尾部不足一字长按字节处理。
- **`memmove`**：需正确处理**重叠区域**：若 `dst < src` 从前往后复制；若 `dst > src` 且区间重叠则从后往前复制。因要处理两种方向，往往比 `memcpy` 稍慢。
- **`memset`**：将区域填充为某个字节值，典型实现扩展单字节到机器字（如重复 8 次形成 64 位值），先按机器字填充再处理边界。
- **`memcmp`**：逐字节或按字比较，遇到差异立即返回第一个不同字节的大小差。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 标准 I/O：printf/scanf/fopen/fread/fwrite

标准 I/O 以 `FILE *` 抽象出"流"，在其之上做 **缓冲 + 格式化 + 线程安全**。

### FILE 结构与缓冲

- `FILE` 内部大致包含：文件描述符 `int fd`、读/写缓冲区的指针/大小/当前位置、文件状态标志（读/写模式、EOF、错误等）、锁（多线程下串行化访问）。
- **缓冲策略**：全缓冲（普通文件，I/O 积累到缓冲区满或显式 `fflush` 才写出）、行缓冲（连接到终端的 `stdout`，遇到换行符刷出）、无缓冲（`stderr` 或显式 `setvbuf` 关闭缓冲）。

### fopen / fclose

- **`fopen`**：解析 `mode` 字符串映射到 `open` 的标志位 → 调用 `open(pathname, flags, mode_bits)` 获取 `fd` → 分配并初始化 `FILE` 结构，分配缓冲区 → 注册到全局流表中。
- **`fclose`**：若缓冲区有未写出数据则调用 `fflush` → 调用 `close(fd)` 关闭文件 → 释放缓冲区和 `FILE` 结构。

### fread / fwrite

- **`fread`**：先检查缓冲区中是否有未消费数据直接拷贝到用户缓冲区；若不足则调用 `read(fd, ...)` 从内核读取新数据填充缓冲区再拷贝；处理 EOF 和错误时设置 `FILE` 标志。
- **`fwrite`**：把用户数据写入 `FILE` 缓冲区；若缓冲区满或显式 `fflush/关闭流` 则调用 `write(fd, ...)` 写入内核；发生错误时设置错误标志。

### printf / fprintf / sprintf

**核心组成：**
- **格式字符串解析器**：识别 `%d/%x/%s/%f/%p` 等占位符及其标志（宽度、精度、对齐、填充等）；
- **数字格式化**：整数支持不同进制、符号、前缀、填充、对齐等；浮点调用内部高精度转换例程（如 `dtoa` 算法），支持舍入、特殊值（NaN/INF）；
- **输出适配器**：`printf` 写入 `stdout`，`fprintf` 写入指定 `FILE*`，`sprintf` 写入用户提供的内存缓冲区（不触发系统调用）。

### scanf / fscanf / sscanf

与 printf 类似但方向相反：解析格式字符串 → 从输入流中按格式读入字符 → 调用解析例程把输入转换为整数/浮点/字符串等，写入调用者提供的指针。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 低级 I/O 与文件描述符封装

- `read`/`write`/`close` 等在 glibc 中通常是对系统调用的一层"薄封装"，主要职责：提供 C 语言调用约定、处理 `errno`（把错误码存入线程本地的 `errno`）、某些平台上提供兼容层。

```c
ssize_t read(int fd, void *buf, size_t count) {
    long ret = __syscall_read(fd, buf, count);  // 内联汇编或 syscall 包装
    if (ret < 0) {
        errno = -ret;
        return -1;
    }
    return (ssize_t)ret;
}
```

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 程序启动与退出：main、exit、atexit

### 程序入口：_start 与 main

- 对于 ELF 可执行文件，入口点是链接器设置的 `_start`（由 C 运行时提供）。
- **_start 的典型工作**：从内核接收栈上的 `argc/argv/envp` → 初始化运行时环境（解析 ELF 动态段完成动态链接、初始化 TLS、构造 `errno` 等全局对象、运行全局构造函数）→ 调用 `main(argc, argv, envp)` → 捕获返回值并调用 `exit(status)` 退出。

### exit / _exit / atexit

- **`exit`**：记录退出码 → 依次调用 `atexit` 注册的清理函数（FILO 顺序）→ 刷新和关闭所有打开的标准 I/O 流 → 释放运行时分配的内部资源 → 最后调用 `_exit(status)` 进入内核。
- **`_exit`**：直接发起系统调用 `exit_group`/`exit`，立即终止进程，不做任何清理。
- **`atexit`**：维护一个函数指针栈，把回调函数指针存入，`exit` 时逆序调用它们。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 进程与环境：system、getenv、setenv

### getenv / setenv / unsetenv

- 环境变量在进程内通常是 `char **environ` 指针数组，每个元素是 `"KEY=VALUE"` 字符串，在进程创建时由内核从父进程复制。
- `getenv`：在 `environ` 中线性扫描匹配对应键名。
- `setenv`：如果存在且允许覆盖则修改，否则重新分配 `"KEY=VALUE"` 字符串并调整 `environ` 数组。
- `unsetenv`：从 `environ` 中移除对应条目并收缩数组。

### system

```c
int system(const char *command);
```

- 若 `command == NULL`：检查是否有可用的 shell，返回非 0/0 表示有/无。
- 否则：调用 `fork()` 创建子进程 → 子进程中调用 `execl("/bin/sh", "sh", "-c", command, (char *)0)` → 父进程调用 `waitpid` 等待子进程结束并返回其退出状态。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 时间与日期：time/clock/gettimeofday/strftime

### time / clock

- **`time`**：典型实现调用系统调用（如 `clock_gettime(CLOCK_REALTIME, ...)` 或老的 `time` syscall），将结果转换为从 Epoch 起的秒数。
- **`clock`**：返回进程消耗的 CPU 时间（用户态+内核态）乘以 `CLOCKS_PER_SEC`，实现往往调用 `getrusage` 或 `clock_gettime(CLOCK_PROCESS_CPUTIME_ID, ...)`。

### gettimeofday / clock_gettime

- 在现代 Linux 上，glibc 更建议使用 `clock_gettime`：内部可能通过 vdso（内核在用户态映射的只读页面）获取时间，避免真正的系统调用；也可能直接发起 `clock_gettime` 系统调用。

### localtime / gmtime / mktime / strftime

- 在秒数与"年月日时分秒 + 时区/夏令时信息"之间做转换，处理闰年、闰秒、本地时区规则（tz database）、夏令时切换等复杂逻辑。大部分逻辑是纯用户态的算术和查表，时区信息通常从 `/usr/share/zoneinfo` 等文件中读取并缓存。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 数学库：libm 的实现思路

数学库（`<math.h>`，如 `sin/cos/exp/log/pow` 等）大多数函数实现为**高精度的数值算法 + 汇编优化**。

- **基本原则**：尽可能满足 C 标准和 IEEE 754 对精度、舍入、异常标志的要求；对普通输入快速、对极端输入保持稳定；注意域错误、溢出、下溢、NaN、无穷大等情况。
- **典型技巧**：范围缩减（range reduction，如 `sin(x)` 把 x 归约到 `[-π/4, π/4]` 再用多项式逼近）、多项式/有理函数逼近（Chebyshev 多项式、minimax 逼近等）、多精度中间结果避免累计误差、对特殊值单独分支。
- **硬件支持**：有些架构提供硬件指令（如 `fsin`/`fcos`），但常常不满足精度要求，现代 libm 往往倾向于使用**软件算法 + SIMD 优化**。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 错误处理：errno 与线程安全

### errno 的本质

- 表面上是一个全局整型变量，但在现代实现中通常是一个**线程局部变量（TLS）**：`errno` 实际是一个宏，展开为 `(*__errno_location())`，`__errno_location()` 返回当前线程私有的 `int *` 指针。
- 系统调用返回负错误码时，标准库封装层会把 `-ret` 转为正的标准错误码（如 `EINTR`、`EAGAIN` 等），写入当前线程的 `errno`。

### 线程安全的库函数

- 许多早期接口本身不是线程安全的（如 `strtok` 使用静态内部状态），现代库提供了带 `_r` 后缀的可重入版本（`strtok_r`、`gmtime_r` 等）。
- 设计新接口时，多采用**显式传入状态指针**或使用 TLS 避免全局共享状态。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## 小结

- C 标准库的大部分代码都运行在用户态，通过内部数据结构、缓冲和算法提供丰富的 API；
- 真正与内核交互的地方相对集中：文件 I/O、内存管理、进程/线程、时间等；
- 整体结构：**"系统调用封装 + 缓冲/抽象 + 兼容性层 + 高级算法"** 是共同的核心模式。

[src: raw/ingested/2技术/cpp/C标准库手册.md]

## Related Pages
- [[内存管理]]
- [[C++语言特性]]
- [[C++进阶知识点]]

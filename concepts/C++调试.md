# C++ 调试

See also: [[C++语言特性]], [[C++多线程与并发]], [[内存管理]]

## 一、GDB 基础与常用命令

### 1.1 常用命令速查表

| 命令 | 缩写 | 说明 |
|------|------|------|
| list | l | 显示多行源代码 |
| break fun / break N | b | 设置断点（函数/行号） |
| break file.c:N | b file.c:N | 在指定文件第 N 行设置断点 |
| break classA::fun | b classA::fun | 在类成员函数处设置断点 |
| break if condition | b if | 条件断点 |
| delete N | d N | 删除断点 |
| disable / enable | - | 禁用/启用断点 |
| info | i | 描述程序状态（i b 断点，i th 线程） |
| run | r | 开始运行程序 |
| display | disp | 每次停止时自动显示变量值 |
| print | p | 打印变量值 |
| watch | - | 监视变量值变化 |
| step N | s N | 单步进入函数 |
| next | n | 单步跳过函数 |
| continue | c | 继续执行到下一断点 |
| finish | - | 执行完当前函数并返回 |
| backtrace | bt | 查看函数调用栈 |
| frame N | f N | 切换栈帧 |
| start | st | 在 main 第一条语句前停下 |
| quit | q | 退出 GDB |
| file | - | 装入需要调试的程序 |
| kill | k | 终止正在调试的程序 |

### 1.2 断点管理

```bash
# 在函数入口设置断点
(gdb) break func1

# 在特定文件行设置断点
(gdb) break breakpoint_demo.c:15

# 条件断点
(gdb) break func2 if y == 3

# 查看所有断点
(gdb) info breakpoints

# 禁用/启用/删除断点
(gdb) disable 2
(gdb) enable 2
(gdb) delete 2

# 修改断点条件
(gdb) condition 1 x > 2
```

### 1.3 变量监视与打印

```bash
# 监视变量变化
(gdb) watch array[3]
(gdb) watch node1.data

# 每次停止时自动显示
(gdb) display array[i]
(gdb) display sum

# 打印变量
(gdb) print array
(gdb) print *node2
(gdb) print node2->next->data

# 查看变量类型
(gdb) whatis node1
```

### 1.4 调用栈分析

```bash
# 显示完整调用栈
(gdb) backtrace full

# 切换栈帧
(gdb) frame 1
(gdb) up
(gdb) down

# 查看特定栈帧详情
(gdb) info frame 2
```

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## 二、多线程调试

### 2.1 GDB 线程调试命令

```bash
# 显示所有线程
info threads

# 切换到指定线程
thread ID
# 或
t 15

# 在所有线程的某行设置断点
break file.c:123 thread all

# 让指定线程执行命令
thread apply ID1 ID2 command

# 让所有线程执行命令
thread apply all bt

# 调度器锁定（单步时只运行当前线程）
set scheduler-locking off|on|step
# off: 不锁定
# on: 只有当前线程执行
# step: 单步时只有当前线程执行
```

### 2.2 进程与线程查找工具

**pgrep**：通过程序名查询进程

```bash
pgrep css
# 15978

pgrep -lo envoy
# 3271 envoy
```

**pstack**：打印进程所有线程的调用栈

```bash
pstack $(pgrep css)
# 或
pstack 15978
```

**top**：查看进程下所有线程

```bash
top -H -p <pid>
```

### 2.3 附着调试运行中的进程

```bash
# 查找目标进程
ps -Tef | grep 程序名

# GDB 附着到进程
gdb attach <pid>
# 或
gdb -p <pid>

# 挂载二进制并附着
gdb path/to/binary <pid>

# 断开调试
detach
```

### 2.4 死锁调试实战

**步骤 1：使用 pstack 初步判断**

程序卡住时，运行 `pstack <pid>`，若多个线程都在 `pthread_mutex_lock` 或 `__lll_lock_wait` 中阻塞，则可能存在死锁。

**步骤 2：GDB 附加并切换线程**

```bash
gdb attach <pid>
(gdb) info threads
(gdb) thread 2
(gdb) bt
(gdb) thread 3
(gdb) bt
```

**步骤 3：查看锁状态**

```gdb
(gdb) frame 3
(gdb) p cc->mutex1
(gdb) p cc->mutex2
```

通过 `__owner` 字段可判断锁被哪个线程持有，从而分析死锁环路。

**死锁示例代码**：

```c
// 线程1: lock(mutex1) -> lock(mutex2)
// 线程2: lock(mutex2) -> lock(mutex1)
// 形成循环等待
```

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## 三、Core Dump 与崩溃分析

### 3.1 启用 Core Dump

```bash
ulimit -c unlimited
echo "/tmp/core.%e.%p" > /proc/sys/kernel/core_pattern
```

### 3.2 使用 GDB 加载 Core 文件

```bash
gdb ./program core
# 或指定 core 路径
gdb ./program /tmp/core.program.12345
```

### 3.3 分析步骤

```bash
# 1. 查看调用栈
(gdb) bt
(gdb) where

# 2. 切换栈帧
(gdb) frame 0
(gdb) f 1

# 3. 查看局部变量
(gdb) info locals

# 4. 打印变量
(gdb) print var_name

# 5. 查看所有线程栈
(gdb) thread apply all bt full
```

### 3.4 信号捕获与 backtrace

在程序中捕获崩溃信号并打印堆栈：

```c
#include <signal.h>
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>

void handle_segv(int signum) {
    void *array[100];
    size_t size;
    char **strings;
    size_t i;

    signal(signum, SIG_DFL);
    size = backtrace(array, 100);
    strings = backtrace_symbols(array, size);

    for (i = 0; i < size; i++) {
        printf("%s\n", strings[i]);
    }
    free(strings);
    exit(1);
}

int main() {
    signal(SIGSEGV, handle_segv);
    int *p = NULL;
    *p = 42;  // 引发段错误
    return 0;
}
```

### 3.5 条件断点辅助复现

若问题难以复现，可设置条件断点：

```bash
(gdb) break function_name if condition
```

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## 四、辅助调试工具

### 4.1 addr2line：地址转源码行

将崩溃地址转换为源代码位置：

```bash
# 基本用法
addr2line -e program 0x400506

# 带函数名和 C++ 符号
addr2line -Cif -e program 0x400afa
```

**示例**：dmesg 显示 `ip:400506`，则：

```bash
addr2line -e test1 400506
# 输出: /path/to/test1.c:5
```

**配合 backtrace**：从 backtrace 输出中提取地址，逐个用 addr2line 解析：

```bash
addr2line -Cif -e test 0x400afa
# FuncBadBoy
# /root/prog/src/test2/test.c:36
```

### 4.2 c++filt：C++ 符号重整

将编译器生成的 mangled 符号还原为可读形式：

```bash
c++filt _ZNK4Json5ValueixEPKc
# 输出: Json::Value::operator[](char const*) const
```

**典型场景**：

- 程序崩溃或链接错误时，日志中出现 `_ZNK4Json5ValueixEPKc` 等符号
- 动态库调试：`ldd -r test.so` 显示未解析符号，用 c++filt 还原

```bash
ldd -r test.so
# 显示 undefined symbol: _ZNK4Json5ValueixEPKc
c++filt _ZNK4Json5ValueixEPKc
```

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## 五、栈破坏与常见问题

### 5.1 局部变量赋值越界

```c
void test() {
    char buff[4] = {0};
    strcpy(buff, "Hello,...");  // 越界写入，破坏栈
}
```

**原因**：`strcpy` 不检查边界，超出数组大小的写入会覆盖栈上其他数据。

**建议**：使用 `strncpy`、`snprintf` 或 `std::string`。

### 5.2 指向局部变量的指针越界

```c
void test(int data) {
    int* p = &data;
    --p;       // 指针越界
    *p = 100;  // 修改未预期内存，破坏栈
}
```

**建议**：避免对栈上指针做越界运算和写入。

### 5.3 死循环导致栈溢出

```c
void test() {
    while (1) {
        int data = 10;  // 每次循环压栈，最终栈溢出
    }
}
```

**建议**：检查循环终止条件，避免无限递归或无限循环中大量局部变量。

### 5.4 总结

| 类型 | 原因 |
|------|------|
| 局部变量越界 | 超出数组边界的写操作覆盖栈 |
| 指针越界 | 指针运算修改栈上未预期区域 |
| 死循环 | 栈空间持续增长导致溢出 |

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## 六、高级调试场景

### 6.1 动态附着调试

```bash
ps -Tef | grep complex_thread
gdb attach 12345

(gdb) info threads
(gdb) break worker_thread
(gdb) continue
(gdb) detach
(gdb) quit
```

### 6.2 GDB 生成 Core 文件

```bash
(gdb) gcore core_name
```

### 6.3 进程与寄存器信息

```bash
(gdb) info proc      # 进程信息
(gdb) info reg       # 寄存器信息
(gdb) info sharedlibrary  # 共享库信息
```

### 6.4 GDB 脚本自动化

创建 `debug_script.gdb`：

```
set pagination off
file complex_thread
break main
run
break worker_thread
commands
  print shared_counter
  continue
end
break monitor_thread
commands
  bt
  continue
end
continue
```

运行：

```bash
gdb -x debug_script.gdb
```

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## 附录：调试流程速查

| 场景 | 工具/命令 |
|------|-----------|
| 程序崩溃 | `ulimit -c unlimited` → `gdb ./prog core` → `bt` |
| 运行中卡住 | `pstack <pid>` → `gdb attach <pid>` → `info threads` → `thread apply all bt` |
| 死锁 | `pstack` 看是否都在 lock → `gdb attach` → 各线程 `bt` → 查锁 `__owner` |
| 地址转行号 | `addr2line -e prog <addr>` |
| C++ 符号还原 | `c++filt <mangled_symbol>` |
| 查找进程 | `pgrep`、`ps -Tef` |
| 查看线程 | `top -H -p <pid>` |

[src: raw/ingested/2技术/cpp/C++调试手册.md]

## Related Pages
- [[C++语言特性]]
- [[C++多线程与并发]]
- [[内存管理]]

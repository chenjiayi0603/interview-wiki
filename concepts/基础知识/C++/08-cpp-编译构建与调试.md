# 编译构建与调试

> CMake、Makefile、编译选项、GDB、Core Dump、addr2line —— 从构建到调试全流程。

---

## 一、构建系统

### 1.1 编译流程

```
源代码(.cpp) → 预处理(.i) → 编译(.s) → 汇编(.o) → 链接 → 可执行文件
```

### 1.2 Makefile 基础

```makefile
# 变量
CXX = g++
CXXFLAGS = -Wall -std=c++17 -O2
LDFLAGS = -lpthread

# 自动变量
# $@ 目标文件名  $< 第一个依赖  $^ 所有依赖

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

$(TARGET): $(OBJS)
	$(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)

.PHONY: clean
clean:
	rm -f $(OBJS) $(TARGET)

# 自动生成头文件依赖
DEPS = $(OBJS:.o=.d)
%.o: %.cpp
	$(CXX) $(CXXFLAGS) -MMD -MP -c $< -o $@
-include $(DEPS)
```

### 1.3 CMake（现代 C++ 标准）

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(myapp main.cpp utils.cpp)

# 包含目录
target_include_directories(myapp PRIVATE include/)

# 链接库
target_link_libraries(myapp PRIVATE pthread)

# 查找外部包
find_package(Boost REQUIRED COMPONENTS system filesystem)
target_link_libraries(myapp PRIVATE Boost::system Boost::filesystem)
```

**CMake 最佳实践**：
- 使用 `target_*` 命令，避免全局命令
- 明确 `PUBLIC / PRIVATE / INTERFACE` 作用域
- 使用 `find_package` 管理外部依赖
- 分离源码和构建目录（out-of-source build）

### 1.4 常用编译选项

```bash
# GCC/Clang
-std=c++17        # C++ 标准
-Wall -Wextra     # 警告
-O2               # 优化
-g                # 调试信息
-fPIC             # 位置无关代码（共享库必需）
-flto             # 链接时优化（LTO）

# MSVC
/std:c++17        # C++ 标准
/W4               # 警告级别
/O2               # 优化
/Zi               # 调试信息
```

### 1.5 构建优化

```bash
# 并行构建
make -j$(nproc)
cmake --build . --parallel 8

# 缓存编译（ccache）
export CC="ccache gcc"
export CXX="ccache g++"

# 预编译头
target_precompile_headers(mylib PRIVATE <vector> <string>)

# 链接时优化
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
```

---

## 二、调试

### 2.1 GDB 常用命令

| 命令 | 缩写 | 说明 |
|------|------|------|
| `list` | l | 显示源代码 |
| `break func / break N` | b | 设置断点 |
| `break if condition` | b if | 条件断点 |
| `run` | r | 运行程序 |
| `print` | p | 打印变量 |
| `display` | disp | 自动显示变量 |
| `step` | s | 单步进入函数 |
| `next` | n | 单步跳过函数 |
| `continue` | c | 继续执行 |
| `backtrace` | bt | 查看调用栈 |
| `frame N` | f N | 切换栈帧 |
| `info threads` | i th | 查看所有线程 |
| `thread apply all bt` | - | 所有线程调用栈 |

### 2.2 多线程调试

```gdb
info threads                      # 显示所有线程
thread 2                          # 切换到线程 2
break file.c:123 thread all       # 所有线程某行设断点
thread apply all bt               # 所有线程栈

set scheduler-locking on|off|step  # 锁定调度器（单步调试用）
```

### 2.3 Core Dump 分析

```bash
# 启用 core dump
ulimit -c unlimited
echo "/tmp/core.%e.%p" > /proc/sys/kernel/core_pattern

# 分析 core 文件
gdb ./program /tmp/core.program.12345
(gdb) bt              # 查看崩溃栈
(gdb) frame 0         # 切换到崩溃帧
(gdb) info locals     # 查看局部变量
(gdb) p var_name      # 打印变量
(gdb) thread apply all bt full  # 所有线程栈 + 局部变量
```

### 2.4 死锁调试流程

1. `pstack <pid>` 看是否都在 `pthread_mutex_lock` 阻塞
2. `gdb attach <pid>` → `info threads` → 各线程 `bt`
3. 查看锁的 `__owner` 字段，分析死锁环路

### 2.5 辅助工具

```bash
# addr2line：地址转源码行
addr2line -Cif -e program 0x400afa
# 输出：函数名 + 文件名:行号

# c++filt：C++ 符号还原
c++filt _ZNK4Json5ValueixEPKc
# 输出：Json::Value::operator[](char const*) const

# pstack：打印线程栈
pstack <pid>

# top -H：查看线程级 CPU
top -H -p <pid>
```

### 2.6 GDB 脚本自动化

```bash
# debug_script.gdb
set pagination off
file myapp
break main
run
break worker_thread
commands
  print counter
  continue
end
continue

# 运行
gdb -x debug_script.gdb
```

---

## 三、编译链接常见问题

### 3.1 静态库 vs 动态库

| 特性 | 静态库（.a/.lib） | 动态库（.so/.dll） |
|------|-------------------|-------------------|
| 链接时机 | 编译时 | 运行时 |
| 可执行文件 | 较大 | 较小 |
| 更新 | 需重新编译 | 替换库文件 |
| 内存 | 每进程一份 | 可共享 |

```bash
# 创建静态库
ar rcs libmylib.a mylib.o

# 创建动态库
g++ -shared -fPIC mylib.cpp -o libmylib.so

# 运行时库路径
export LD_LIBRARY_PATH=/path/to/lib:$LD_LIBRARY_PATH
```

### 3.2 栈破坏常见原因

| 类型 | 原因 |
|------|------|
| 局部变量越界 | `strcpy` 不检查边界 |
| 指针越界 | 对栈上指针做越界运算 |
| 死循环/深递归 | 栈空间持续增长导致溢出 |

---

## 四、面试高频追问

### Q1: 静态链接 vs 动态链接优缺点？
| 对比 | 静态链接 | 动态链接 |
|:----:|:--------:|:--------:|
| 启动速度 | 快 | 慢（需 ld.so 解析符号） |
| 二进制体积 | 大 | 小 |
| 部署 | 单文件 | 需携带 .so |
| 安全更新 | 需重新编译 | 替换 .so 即可 |
| **低延迟推荐** | ✅ | ❌（启动抖动） |

### Q2: CMake 中 Debug/Release 构建类型区别？
- **Debug**：`-g -O0`，关闭优化，保留符号，方便调试
- **Release**：`-O3 -DNDEBUG`，最高优化，去除断言
- **RelWithDebInfo**：`-O2 -g`，优化中带符号
- **MinSizeRel**：`-Os`，体积优化

### Q3: GDB 常用调试命令？
```bash
break main          # 打断点
run                 # 运行
next / step         # 下一步 / 步入
continue            # 继续到下一个断点
print var           # 打印变量
backtrace           # 查看调用栈
info locals         # 查看所有局部变量
frame N             # 切换到第 N 层栈帧
watch var           # 监视变量变化
```

### Q4: Core Dump 如何调试？
```bash
ulimit -c unlimited            # 开启 core dump
gdb ./program core.12345       # 加载 core 文件
bt                             # 查看崩溃时的调用栈
info registers                 # 查看寄存器状态
frame N                        # 定位到崩溃帧
```

### Q5: LTO 为什么能提升性能？
- LTO 在链接时进行跨编译单元优化
- 展开跨文件的内联、常量传播、死代码消除
- 模板实例化可以跨文件合并，减少冗余
- **代价**：编译时间增加 20-50%，内存消耗翻倍

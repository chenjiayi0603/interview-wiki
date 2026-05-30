# Sanitizer 工具详解

## 2.1 AddressSanitizer (ASan)

### 2.1.1 基本使用

**AddressSanitizer 特点**：
- 由 Google 开发，集成到 GCC/Clang 编译器
- 速度快（比 Valgrind 快 2-3 倍）
- 检测内存越界、使用已释放内存、重复释放等问题

**编译选项**：
```bash
# 基本使用
g++ -fsanitize=address -g -O1 program.cpp -o program

# 启用所有检测
g++ -fsanitize=address -fsanitize=undefined -g -O1 program.cpp -o program

# 生成符号化报告
g++ -fsanitize=address -g -O1 -fno-omit-frame-pointer program.cpp -o program
```

**各命令详细作用**：

- `g++ -fsanitize=address -g -O1 program.cpp -o program`
  - 开启 **AddressSanitizer (ASan)**，重点检测内存相关问题：越界访问、Use-After-Free、Double Free、栈/堆溢出等。
  - `-g` 用于生成调试符号，报错时可直接定位到源码行号与函数栈。
  - `-O1` 在可读栈信息和运行效率之间做平衡，适合日常开发调试。

- `g++ -fsanitize=address -fsanitize=undefined -g -O1 program.cpp -o program`
  - 在 ASan 基础上叠加 **UndefinedBehaviorSanitizer (UBSan)**。
  - 除内存问题外，还可检测常见未定义行为：有符号整型溢出、非法移位、空指针解引用、对齐错误等。
  - 适合测试阶段做更全面的质量扫描。

- `g++ -fsanitize=address -g -O1 -fno-omit-frame-pointer program.cpp -o program`
  - `-fno-omit-frame-pointer` 强制保留栈帧指针，使调用栈回溯更稳定、完整。
  - 当崩溃链路较深或优化导致栈信息不完整时，该选项能显著提升定位效率。
  - 适合线上问题复现与根因分析场景。

**实战建议**：

- Sanitizer 需要在程序运行时触发对应路径才会报错，建议配合单测/压测扩大覆盖面。
- 开发调试优先用 `-O1`；若需要更可读的调用栈，可临时改为 `-O0`。
- 常见运行参数（按需开启）：
  - `ASAN_OPTIONS=detect_leaks=1`：启用内存泄漏检测（Linux 常用）。
  - `ASAN_OPTIONS=halt_on_error=1`：首次错误即中断，便于首错定位。

### 2.1.2 检测内存越界

**示例代码**：
```cpp
// asan_overflow.cpp
#include <iostream>

int main() {
    int arr[10];
    arr[10] = 42;  // 错误：越界访问
    std::cout << arr[10] << std::endl;
    return 0;
}
```

**编译和运行**：
```bash
g++ -fsanitize=address -g -O1 asan_overflow.cpp -o asan_overflow
./asan_overflow
```

**输出示例**：
```
=================================================================
==12345==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fff12345678
==12345==WRITE of size 4 at 0x7fff12345678 thread T0
    #0 0x4005f6 in main asan_overflow.cpp:5
    #1 0x7f1234567890 in __libc_start_main
    #2 0x4004e0 in _start

Address 0x7fff12345678 is located in stack of thread T0 at offset 40 in frame
    #0 0x4005e0 in main asan_overflow.cpp:4

SUMMARY: AddressSanitizer: stack-buffer-overflow asan_overflow.cpp:5 in main
```

### 2.1.3 检测使用已释放内存

**示例代码**：
```cpp
// asan_uaf.cpp
#include <iostream>

int main() {
    int* p = new int(42);
    delete p;
    *p = 100;  // 错误：使用已释放的内存
    std::cout << *p << std::endl;
    return 0;
}
```

**输出示例**：
```
=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x602000000010
==12345==WRITE of size 4 at 0x602000000010 thread T0
    #0 0x4005f6 in main asan_uaf.cpp:6
    #1 0x7f1234567890 in __libc_start_main

0x602000000010 is located 0 bytes inside of 4-byte region [0x602000000010,0x602000000014)
freed by thread T0 here:
    #0 0x7f1234567890 in operator delete(void*)
    #1 0x4005e1 in main asan_uaf.cpp:5

SUMMARY: AddressSanitizer: heap-use-after-free asan_uaf.cpp:6 in main
```

### 2.1.4 检测重复释放（double-free）

**示例代码**：
```cpp
// asan_double_free.cpp
#include <iostream>

int main() {
    int* p = new int(42);
    delete p;
    delete p;  // 错误：重复释放
    return 0;
}
```

**编译和运行**：
```bash
g++ -fsanitize=address -g -O1 asan_double_free.cpp -o asan_double_free
./asan_double_free
```

**输出示例**：
```
=================================================================
==12345==ERROR: AddressSanitizer: attempting double-free on 0x602000000010
==12345==    #0 0x7f1234567890 in operator delete(void*)
==12345==    #1 0x4005f6 in main asan_double_free.cpp:7
==12345==
==12345==Address 0x602000000010 is located 0 bytes inside of 4-byte region [0x602000000010,0x602000000014)
==12345==freed by thread T0 here:
==12345==    #0 0x7f1234567890 in operator delete(void*)
==12345==    #1 0x4005e1 in main asan_double_free.cpp:6
==12345==
SUMMARY: AddressSanitizer: double-free asan_double_free.cpp:7 in main
```

### 2.1.5 ASan 常用选项

**环境变量控制**：
```bash
# 设置选项
export ASAN_OPTIONS=detect_leaks=1:halt_on_error=0:abort_on_error=1

# 运行程序
./program
```

**常用选项**：
- `detect_leaks=1`：启用泄漏检测（ASan 内置 LSan）
- `halt_on_error=0`：不立即停止，继续检测
- `abort_on_error=1`：检测到错误时中止
- `print_stats=1`：打印统计信息
- `verbosity=1`：详细输出

**一行完整示例**：
```bash
ASAN_OPTIONS=detect_leaks=1:abort_on_error=1 ./asan_overflow
```

---

## 2.2 LeakSanitizer (LSan)

### 2.2.1 基本使用

**LeakSanitizer 特点**：
- 专门用于检测内存泄漏
- 通常集成在 AddressSanitizer 中
- 也可以独立使用

**编译选项**：
```bash
# 独立使用 LeakSanitizer
g++ -fsanitize=leak -g program.cpp -o program

# 或使用 AddressSanitizer（包含 LSan）
g++ -fsanitize=address -g -O1 program.cpp -o program
```

### 2.2.2 检测内存泄漏

**示例代码**：
```cpp
// lsan_leak.cpp
#include <iostream>

void memory_leak() {
    int* p = new int(42);
    // 忘记 delete p;
}

int main() {
    memory_leak();
    return 0;
}
```

**编译和运行**：
```bash
g++ -fsanitize=leak -g lsan_leak.cpp -o lsan_leak
./lsan_leak
```

**输出示例**：
```
=================================================================
==12345==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 4 byte(s) in 1 object(s) allocated from:
    #0 0x7f1234567890 in operator new(unsigned long)
    #1 0x4005f6 in memory_leak() lsan_leak.cpp:4
    #2 0x400611 in main lsan_leak.cpp:8

SUMMARY: LeakSanitizer: 4 byte(s) leaked in 1 allocation(s).
```

### 2.2.3 LSan 常用选项

**环境变量控制**：
```bash
export LSAN_OPTIONS=detect_leaks=1:print_stats=1:verbosity=1
./program
```

**独立 LSan 完整示例（不依赖 ASan）**：
```bash
# 仅检测泄漏，不检测越界/UAF
g++ -fsanitize=leak -g -O0 lsan_leak.cpp -o lsan_leak
LSAN_OPTIONS=verbosity=1 ./lsan_leak
```

---

## 2.3 MemorySanitizer (MSan)

### 2.3.1 基本使用

**MemorySanitizer 特点**：
- 专门检测未初始化内存的使用
- 需要所有代码都用 MSan 编译（包括库）

**编译选项**：
```bash
# 基本使用
g++ -fsanitize=memory -g -O1 program.cpp -o program

# 需要链接 MSan 运行时
g++ -fsanitize=memory -g -O1 -fno-omit-frame-pointer program.cpp -o program
```

### 2.3.2 检测未初始化内存

**示例代码**：
```cpp
// msan_uninit.cpp
#include <iostream>

int main() {
    int x;  // 未初始化
    std::cout << x << std::endl;  // 错误：使用未初始化的变量
    return 0;
}
```

**编译和运行**：
```bash
# Clang 对 MSan 支持更好，GCC 需注意 libc 等需用 MSan 编译
clang++ -fsanitize=memory -g -O1 msan_uninit.cpp -o msan_uninit
# 或 gcc（若系统有 MSan 运行时）
g++ -fsanitize=memory -g -O1 msan_uninit.cpp -o msan_uninit
./msan_uninit
```

**输出示例**：
```
==12345==WARNING: MemorySanitizer: use-of-uninitialized-value
    #0 0x4005f6 in main msan_uninit.cpp:5
    #1 0x7f1234567890 in __libc_start_main

SUMMARY: MemorySanitizer: use-of-uninitialized-value msan_uninit.cpp:5 in main
```

---

## 2.4 ThreadSanitizer (TSan)

### 2.4.1 基本使用

**ThreadSanitizer 特点**：
- 检测数据竞争（Data Race）
- 检测死锁
- 多线程程序调试必备

**编译选项**：
```bash
# 基本使用
g++ -fsanitize=thread -g -O1 program.cpp -o program -pthread
```

### 2.4.2 检测数据竞争

**示例代码**：
```cpp
// tsan_race.cpp
#include <thread>
#include <iostream>

int counter = 0;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        counter++;  // 数据竞争
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << counter << std::endl;
    return 0;
}
```

**编译和运行**：
```bash
g++ -fsanitize=thread -g -O1 tsan_race.cpp -o tsan_race -pthread
./tsan_race
```

**输出示例**：
```
==================
WARNING: ThreadSanitizer: data race
  Write of size 4 at 0x000000601070 by thread T2:
    #0 increment() tsan_race.cpp:7
    #1 <null> <null>

  Previous write of size 4 at 0x000000601070 by thread T1:
    #0 increment() tsan_race.cpp:7
    #1 <null> <null>

SUMMARY: ThreadSanitizer: data race tsan_race.cpp:7 in increment()
```

**正确写法示例（避免数据竞争）**：
```cpp
// tsan_fixed.cpp：使用互斥锁
#include <thread>
#include <iostream>
#include <mutex>

int counter = 0;
std::mutex mtx;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        counter++;
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << counter << std::endl;  // 200000
    return 0;
}
```
```bash
g++ -fsanitize=thread -g -O1 tsan_fixed.cpp -o tsan_fixed -pthread
./tsan_fixed   # 无 TSan 报告
```

---

## 2.5 Sanitizer 工具对比

| 工具 | 检测问题 | 性能影响 | 适用场景 |
|------|---------|---------|---------|
| **AddressSanitizer** | 内存越界、UAF、重复释放 | 2-3倍 | 日常开发，快速检测 |
| **LeakSanitizer** | 内存泄漏 | 2-3倍 | 快速泄漏检测 |
| **MemorySanitizer** | 未初始化内存 | 3-5倍 | 未初始化内存问题 |
| **ThreadSanitizer** | 数据竞争、死锁 | 5-15倍 | 多线程调试 |

**Sanitizer vs Valgrind**：

| 特性 | Sanitizer | Valgrind |
|------|-----------|----------|
| **速度** | 快（2-3倍） | 慢（10-50倍） |
| **集成** | 编译器集成 | 独立工具 |
| **功能** | 特定问题检测 | 全面检测 |
| **适用场景** | 日常开发 | 深度调试 |

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-2.-Sanitizer-工具详解.md]
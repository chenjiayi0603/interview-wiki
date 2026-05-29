# Valgrind 其他工具

## 3.1 Callgrind（函数调用分析）

**Callgrind 用途**：
- 分析函数调用关系
- 统计函数调用次数
- 分析函数执行时间
- 生成调用图

**基本命令**：
```bash
# 基本使用
valgrind --tool=callgrind ./program

# 指定输出文件
valgrind --tool=callgrind --callgrind-out-file=callgrind.out ./program

# 查看报告
callgrind_annotate callgrind.out.12345

# 或使用图形化工具
kcachegrind callgrind.out.12345
```

---

## 3.2 Helgrind（线程错误检测）

**Helgrind 用途**：
- 检测数据竞争（Data Race）
- 检测死锁
- 检测锁顺序问题
- 多线程程序调试必备

**基本命令**：
```bash
# 基本使用
valgrind --tool=helgrind ./program

# 指定输出文件
valgrind --tool=helgrind --log-file=helgrind.log ./program
```

**检测示例**：
```cpp
// race_condition.cpp
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

**编译和检测**：
```bash
g++ -g -O0 -pthread race_condition.cpp -o race_condition
valgrind --tool=helgrind ./race_condition
```

---

## 3.3 Cachegrind（缓存性能分析）

**Cachegrind 用途**：
- 分析缓存命中率
- 统计 L1/L2/L3 缓存访问
- 定位缓存不友好的代码
- 优化内存访问模式

**基本命令**：
```bash
# 基本使用
valgrind --tool=cachegrind ./program

# 指定输出文件
valgrind --tool=cachegrind --cachegrind-out-file=cachegrind.out ./program

# 查看报告
cg_annotate cachegrind.out.12345
```

---

## 3.4 Valgrind 工具对比

| 工具 | 用途 | 性能影响 | 适用场景 | 输出工具 |
|------|------|---------|---------|---------|
| **Memcheck** | 内存错误检测 | 慢（10-50倍） | 深度调试，内存问题 | 直接输出 |
| **Massif** | 堆内存分析 | 慢（10-50倍） | 内存占用分析 | `ms_print` |
| **Callgrind** | 函数调用分析 | 慢（10-50倍） | 性能分析，调用关系 | `callgrind_annotate`, `kcachegrind` |
| **Helgrind** | 线程错误检测 | 慢（10-50倍） | 多线程调试 | 直接输出 |
| **Cachegrind** | 缓存性能分析 | 慢（10-50倍） | 缓存命中率分析 | `cg_annotate` |

[src: raw/ingested/2技术/性能优化/内存优化-valgrind-3.-Valgrind-其他工具.md]
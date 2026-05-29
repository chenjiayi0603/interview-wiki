# Valgrind Massif（堆内存分析）

## 2.1 基本使用

### 2.1.1 Massif 用途

- 分析程序的堆内存使用情况
- 找出内存使用峰值和增长点
- 定位内存占用热点
- 优化内存分配策略

### 2.1.2 基本命令

```bash
# 基本使用
valgrind --tool=massif ./program

# 指定输出文件
valgrind --tool=massif --massif-out-file=massif.out ./program

# 查看报告
ms_print massif.out.12345

# 或使用新工具（如果可用）
massif-visualizer massif.out.12345
```

---

## 2.2 示例分析

### 2.2.1 示例代码

```cpp
// heap_usage.cpp
#include <vector>
#include <iostream>

void allocate_memory() {
    std::vector<int> vec1(1000000);  // 分配 4MB
    std::vector<int> vec2(1000000);  // 再分配 4MB
    // vec1 和 vec2 在函数结束时自动释放
}

int main() {
    allocate_memory();
    std::vector<int> vec3(1000000);  // 分配 4MB
    return 0;
}
```

### 2.2.2 检测命令

```bash
g++ -g -O0 heap_usage.cpp -o heap_usage
valgrind --tool=massif --massif-out-file=massif.out ./heap_usage
ms_print massif.out.*
```

### 2.2.3 输出解读

```
    MB
4.000^                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
     |                                                                    #
   0 +----------------------------------------------------------------------->Gi
     0                                                                   100.00

Number of snapshots: 25
 Detailed snapshots: [2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24]
```

**关键信息**：
- 峰值内存使用：4MB
- 内存增长点：可通过详细快照查看
- 内存分配调用栈：可定位具体代码位置

### 2.2.4 详细快照分析

```
--------------------------------------------------------------------------------
  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
--------------------------------------------------------------------------------
 10      1,234,567            4,000            4,000             0            0
 99.90% (4,000B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->99.90% (4,000B) 0x4005F6: allocate_memory() (heap_usage.cpp:5)
```

**关键信息**：
- `total(B)`：总内存使用
- `useful-heap(B)`：实际有用的堆内存
- `extra-heap(B)`：额外开销（如对齐、元数据）
- 调用栈：显示内存分配位置

---

## 2.3 Massif 常用选项

| 选项 | 说明 | 示例 | 使用场景 |
|------|------|------|---------|
| `--massif-out-file=<file>` | 指定输出文件 | `--massif-out-file=massif.out` | 自定义输出文件名 |
| `--time-unit=<i|ms|B>` | 时间单位 | `--time-unit=ms` | 按时间或指令数采样 |
| `--heap=yes` | 只分析堆内存 | `--heap=yes` | 默认开启 |
| `--stacks=yes` | 分析栈内存 | `--stacks=yes` | 需要分析栈内存时 |
| `--pages-as-heap=yes` | 将页分配视为堆 | `--pages-as-heap=yes` | 分析 mmap 分配时 |
| `--detailed-freq=<n>` | 详细快照频率 | `--detailed-freq=10` | 控制详细快照数量 |
| `--max-snapshots=<n>` | 最大快照数 | `--max-snapshots=100` | 限制快照数量 |

[src: raw/ingested/2技术/性能优化/内存优化-valgrind-2.-Valgrind-Massif（堆内存分析）.md]
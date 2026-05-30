# TBB 高级特性

> 本文介绍 Intel TBB（Threading Building Blocks）的高级特性：任务组（Task Group）、分区器（Partitioner）、可组合性（Composability）和任务优先级模拟。

See also: [[TBB最佳实践]], [[C++TBB最佳实践]], [[C++多线程与并发]], [[C++并发性能优化]]

## 一、任务组（Task Group）

`tbb::task_group` 允许动态添加并行任务，并通过 `wait()` 等待所有任务完成。

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <thread>
#include <chrono>
#include <mutex>

std::mutex cout_mutex;  // 用于同步输出

int main() {
    std::cout << "=== Task Group 示例 ===" << std::endl;
    
    tbb::task_group g;
    auto start = std::chrono::high_resolution_clock::now();
    
    // 运行三个并行任务
    g.run([]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        std::lock_guard<std::mutex> lock(cout_mutex);
        std::cout << "任务1完成 (200ms) - 线程ID: " << std::this_thread::get_id() << std::endl;
    });
    
    g.run([]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(150));
        std::lock_guard<std::mutex> lock(cout_mutex);
        std::cout << "任务2完成 (150ms) - 线程ID: " << std::this_thread::get_id() << std::endl;
    });
    
    g.run([]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        std::lock_guard<std::mutex> lock(cout_mutex);
        std::cout << "任务3完成 (100ms) - 线程ID: " << std::this_thread::get_id() << std::endl;
    });
    
    std::cout << "主线程: 等待所有任务完成..." << std::endl;
    g.wait();  // 等待所有任务完成
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "\n所有任务完成！" << std::endl;
    std::cout << "总耗时: " << duration.count() << " ms" << std::endl;
    std::cout << "(串行需要 450ms，并行只需约 200ms)" << std::endl;
    
    return 0;
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-三、高级特性.md]

## 二、分区器（Partitioner）

**目的**：控制 `parallel_for` / `parallel_reduce` 等把迭代范围切成多少块、每块多大，从而在**负载均衡、调度开销、缓存利用**之间做取舍。

**场景与选型**：

| 分区器 | 典型场景 | 为了啥 |
|--------|----------|--------|
| **auto_partitioner**（默认） | 通用循环、事先不知道每次迭代耗时 | 运行时自动调块大小，兼顾负载均衡和调度开销，**大多数情况直接用** |
| **affinity_partitioner** | **同一组数据被同一循环多次执行**（如时间步循环、迭代求解） | 让同一块数据尽量由同一线程处理，**提高缓存命中、减少伪共享** |
| **simple_partitioner** | 每次迭代工作量**差异大**（有的很快、有的很慢） | 切得很细、块多，**便于负载均衡**，调度开销会变大 |
| **static_partitioner** | 每次迭代工作量**几乎相同**、且总工作量足够大 | 按线程数均分，块数少、**调度开销最低**，几乎不做动态均衡 |

**一句话**：默认不指定用 `auto_partitioner`；要缓存友好、多次跑同一循环用 `affinity_partitioner`；负载很不均匀用 `simple_partitioner`；负载均匀、追求低开销用 `static_partitioner`。

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-三、高级特性.md]

## 三、可组合性（Composability）

TBB 支持嵌套并行，可以组合使用不同的并行模式：

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <mutex>

std::mutex cout_mutex;

int main() {
    std::cout << "=== 嵌套并行示例 ===" << std::endl;
    
    // 创建一个 4x5 的矩阵任务
    const int OUTER = 4;
    const int INNER = 5;
    std::vector<std::vector<int>> results(OUTER, std::vector<int>(INNER, 0));
    
    // 外层并行
    tbb::parallel_for(0, OUTER, [&](int i) {
        {
            std::lock_guard<std::mutex> lock(cout_mutex);
            std::cout << "外层任务 " << i << " 开始" << std::endl;
        }
        
        // 内层并行 - TBB 会自动管理线程，避免过度订阅
        tbb::parallel_for(0, INNER, [&](int j) {
            results[i][j] = (i + 1) * (j + 1);  // 计算乘法表
        });
        
        {
            std::lock_guard<std::mutex> lock(cout_mutex);
            std::cout << "外层任务 " << i << " 完成" << std::endl;
        }
    });
    
    // 打印结果矩阵
    std::cout << "\n结果矩阵 (乘法表):" << std::endl;
    for (int i = 0; i < OUTER; ++i) {
        std::cout << "  行" << (i+1) << ": ";
        for (int j = 0; j < INNER; ++j) {
            std::cout << results[i][j] << "\t";
        }
        std::cout << std::endl;
    }
    
    // 演示 parallel_for + parallel_reduce 组合
    std::cout << "\n=== 组合使用 parallel_for 和 parallel_reduce ===" << std::endl;
    
    std::vector<std::vector<int>> matrix(3, std::vector<int>{1, 2, 3, 4, 5});
    std::vector<int> row_sums(3);
    
    // 外层并行计算每行
    tbb::parallel_for(0, 3, [&](int row) {
        // 内层并行归约
        row_sums[row] = tbb::parallel_reduce(
            tbb::blocked_range<size_t>(0, matrix[row].size()),
            0,
            [&](const tbb::blocked_range<size_t>& r, int init) {
                for (size_t i = r.begin(); i < r.end(); ++i) {
                    init += matrix[row][i];
                }
                return init;
            },
            std::plus<int>()
        );
    });
    
    std::cout << "每行之和: ";
    for (int sum : row_sums) {
        std::cout << sum << " ";
    }
    std::cout << "(预期: 15 15 15)" << std::endl;
    
    return 0;
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-三、高级特性.md]

## 四、任务优先级

注意：旧的 `tbb::task` API 在新版 TBB 中已废弃。推荐使用 `task_group` 配合 `task_arena` 来控制任务执行：

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <thread>
#include <chrono>
#include <mutex>
#include <queue>

std::mutex cout_mutex;

int main() {
    std::cout << "=== 任务优先级模拟示例 ===" << std::endl;
    
    // 使用 task_arena 控制并行度
    tbb::task_arena arena(4);  // 使用4个线程
    
    std::atomic<int> high_completed{0};
    std::atomic<int> low_completed{0};
    
    arena.execute([&] {
        tbb::task_group high_priority_group;
        tbb::task_group low_priority_group;
        
        // 先提交低优先级任务
        for (int i = 0; i < 10; ++i) {
            low_priority_group.run([&, i] {
                std::this_thread::sleep_for(std::chrono::milliseconds(50));
                low_completed++;
                std::lock_guard<std::mutex> lock(cout_mutex);
                std::cout << "低优先级任务 " << i << " 完成" << std::endl;
            });
        }
        
        // 后提交高优先级任务（模拟：使用单独的 task_group 优先等待）
        for (int i = 0; i < 5; ++i) {
            high_priority_group.run([&, i] {
                std::this_thread::sleep_for(std::chrono::milliseconds(30));
                high_completed++;
                std::lock_guard<std::mutex> lock(cout_mutex);
                std::cout << ">>> 高优先级任务 " << i << " 完成" << std::endl;
            });
        }
        
        // 先等待高优先级任务
        std::cout << "\n等待高优先级任务..." << std::endl;
        high_priority_group.wait();
        std::cout << "高优先级任务全部完成: " << high_completed << "/5" << std::endl;
        
        // 再等待低优先级任务
        std::cout << "\n等待低优先级任务..." << std::endl;
        low_priority_group.wait();
        std::cout << "低优先级任务全部完成: " << low_completed << "/10" << std::endl;
    });
    
    std::cout << "\n=== 使用 isolate 隔离任务 ===" << std::endl;
    
    // 使用 isolate 确保关键任务优先执行
    tbb::task_group g;
    
    g.run([&] {
        tbb::this_task_arena::isolate([&] {
            // 这个任务会被隔离执行，不会被其他任务偷取
            std::cout << "隔离任务执行中..." << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::cout << "隔离任务完成" << std::endl;
        });
    });
    
    g.wait();
    
    return 0;
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-三、高级特性.md]
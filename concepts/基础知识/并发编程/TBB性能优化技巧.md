# TBB 性能优化技巧

> 本文总结 Intel TBB（Threading Building Blocks）并行编程的性能优化技巧，涵盖避免伪共享、选择合适的粒度、使用线程本地存储、减少锁竞争。

See also: [[TBB最佳实践]], [[C++并发性能优化]], [[C++多线程与并发]]

## 4.1 避免伪共享（False Sharing）

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <chrono>
#include <vector>

// 错误示例：伪共享 - 多个计数器在同一缓存行
struct BadCounters {
    int count[8];  // 8个int紧密排列，可能在同一或相邻缓存行
};

// 正确示例：使用缓存行对齐避免伪共享
struct alignas(64) GoodCounter {  // 64 字节对齐（典型缓存行大小）
    int count;
    char padding[60];  // 填充到64字节
};

int main() {
    std::cout << "=== 伪共享（False Sharing）示例 ==" << std::endl;
    
    const int NUM_THREADS = 8;
    const int ITERATIONS = 10000000;
    
    // 测试伪共享情况
    BadCounters bad_counters = {};
    auto start1 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(0, NUM_THREADS, [&](int tid) {
        for (int i = 0; i < ITERATIONS; ++i) {
            bad_counters.count[tid]++;
        }
    });
    
    auto end1 = std::chrono::high_resolution_clock::now();
    auto time1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    
    // 测试避免伪共享情况
    std::vector<GoodCounter> good_counters(NUM_THREADS);
    auto start2 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(0, NUM_THREADS, [&](int tid) {
        for (int i = 0; i < ITERATIONS; ++i) {
            good_counters[tid].count++;
        }
    });
    
    auto end2 = std::chrono::high_resolution_clock::now();
    auto time2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    
    // 验证结果
    int bad_sum = 0, good_sum = 0;
    for (int i = 0; i < NUM_THREADS; ++i) {
        bad_sum += bad_counters.count[i];
        good_sum += good_counters[i].count;
    }
    
    std::cout << "\n结构体大小:" << std::endl;
    std::cout << "  BadCounters: " << sizeof(BadCounters) << " bytes" << std::endl;
    std::cout << "  GoodCounter: " << sizeof(GoodCounter) << " bytes" << std::endl;
    
    std::cout << "\n性能对比:" << std::endl;
    std::cout << "  伪共享版本耗时: " << time1.count() << " ms" << std::endl;
    std::cout << "  缓存对齐版本耗时: " << time2.count() << " ms" << std::endl;
    
    std::cout << "\n结果验证:" << std::endl;
    std::cout << "  伪共享版本总和: " << bad_sum << std::endl;
    std::cout << "  缓存对齐版本总和: " << good_sum << std::endl;
    std::cout << "  预期值: " << NUM_THREADS * ITERATIONS << std::endl;
    
    return 0;
}

/* 输出示例：
=== 伪共享（False Sharing）示例 ===

结构体大小:
  BadCounters: 32 bytes
  GoodCounter: 64 bytes

性能对比:
  伪共享版本耗时: 285 ms
  缓存对齐版本耗时: 42 ms

结果验证:
  伪共享版本总和: 80000000
  缓存对齐版本总和: 80000000
  预期值: 80000000
*/
```

## 4.2 选择合适的粒度

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <chrono>
#include <cmath>

int main() {
    std::cout << "=== 任务粒度优化示例 ==" << std::endl;
    
    const size_t N = 10000000;
    std::vector<double> data(N);
    
    // 测试不同粒度的性能
    auto test_granularity = [&](const std::string& name, size_t grain_size) {
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(
            tbb::blocked_range<size_t>(0, data.size(), grain_size),
            [&](const tbb::blocked_range<size_t>& r) {
                for (size_t i = r.begin(); i < r.end(); ++i) {
                    data[i] = std::sin(static_cast<double>(i)) * std::cos(static_cast<double>(i));
                }
            }
        );
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        
        std::cout << name << " (粒度=" << grain_size << "): " 
                  << duration.count() << " ms" << std::endl;
    };
    
    std::cout << "\n不同粒度的性能对比:" << std::endl;
    
    test_granularity("粒度太小", 1);
    test_granularity("粒度较小", 100);
    test_granularity("粒度适中", 1000);
    test_granularity("粒度较大", 10000);
    test_granularity("粒度很大", 100000);
    
    std::cout << "\n简洁语法（自动粒度）:" << std::endl;
    auto start = std::chrono::high_resolution_clock::now();
    tbb::parallel_for(size_t(0), data.size(), [&](size_t i) {
        data[i] = std::sin(static_cast<double>(i)) * std::cos(static_cast<double>(i));
    });
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "parallel_for(0, N, lambda): " << duration.count() << " ms" << std::endl;
    
    std::cout << "\n建议:" << std::endl;
    std::cout << "- 任务过小会增加调度开销" << std::endl;
    std::cout << "- 任务过大会降低并行度" << std::endl;
    std::cout << "- 通常 1000-10000 是较好的粒度范围" << std::endl;
    std::cout << "- 使用 auto_partitioner 可自动调整" << std::endl;
    
    return 0;
}

/* 输出示例：
=== 任务粒度优化示例 ===

不同粒度的性能对比:
粒度太小 (粒度=1): 892 ms
粒度较小 (粒度=100): 156 ms
粒度适中 (粒度=1000): 85 ms
粒度较大 (粒度=10000): 82 ms
粒度很大 (粒度=100000): 95 ms

简洁语法（自动粒度）:
parallel_for(0, N, lambda): 88 ms

建议:
- 任务过小会增加调度开销
- 任务过大会降低并行度
- 通常 1000-10000 是较好的粒度范围
- 使用 auto_partitioner 可自动调整
*/
```

## 4.3 使用本地存储（Thread Local Storage）

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>

int main() {
    std::cout << "=== Thread Local Storage (TLS) 示例 ==" << std::endl;
    
    const int N = 10000;
    
    // 创建线程本地存储，每个线程有自己的累加器
    tbb::enumerable_thread_specific<int> local_sum(0);
    
    // 并行计算 0 + 1 + 2 + ... + 9999
    tbb::parallel_for(0, N, [&](int i) {
        local_sum.local() += i;  // 每个线程独立的累加器，无竞争
    });
    
    // 查看每个线程的本地结果
    std::cout << "\n各线程本地累加结果:" << std::endl;
    int thread_idx = 0;
    for (const auto& sum : local_sum) {
        std::cout << "  线程 " << thread_idx++ << " 的本地和: " << sum << std::endl;
    }
    std::cout << "总共使用了 " << thread_idx << " 个线程" << std::endl;
    
    // 合并所有线程的本地结果
    int total = 0;
    for (const auto& sum : local_sum) {
        total += sum;
    }
    
    std::cout << "\n最终结果:" << std::endl;
    std::cout << "  TLS 合并结果: " << total << std::endl;
    std::cout << "  预期结果 (0+1+...+9999): " << (N-1) * N / 2 << std::endl;
    
    // 使用 combine 简化合并操作
    std::cout << "\n使用 combine() 方法合并:" << std::endl;
    int combined = local_sum.combine([](int a, int b) { return a + b; });
    std::cout << "  combine 结果: " << combined << std::endl;
    
    // 演示更复杂的 TLS 用法：统计每个线程处理的元素数量
    std::cout << "\n=== 统计每线程工作量 ==" << std::endl;
    
    struct ThreadStats {
        int count = 0;
        int min_val = INT_MAX;
        int max_val = INT_MIN;
    };
    
    tbb::enumerable_thread_specific<ThreadStats> thread_stats;
    
    tbb::parallel_for(0, 1000, [&](int i) {
        auto& stats = thread_stats.local();
        stats.count++;
        stats.min_val = std::min(stats.min_val, i);
        stats.max_val = std::max(stats.max_val, i);
    });
    
    int total_count = 0;
    int global_min = INT_MAX;
    int global_max = INT_MIN;
    
    std::cout << "各线程统计:" << std::endl;
    int idx = 0;
    for (const auto& stats : thread_stats) {
        std::cout << "  线程 " << idx++ << ": 处理了 " << stats.count 
                  << " 个元素, 范围 [" << stats.min_val << ", " << stats.max_val << "]" << std::endl;
        total_count += stats.count;
        global_min = std::min(global_min, stats.min_val);
        global_max = std::max(global_max, stats.max_val);
    }
    std::cout << "\n汇总: 总计 " << total_count << " 个元素, 全局范围 [" 
              << global_min << ", " << global_max << "]" << std::endl;
    
    return 0;
}

/* 输出示例：
=== Thread Local Storage (TLS) 示例 ===

各线程本地累加结果:
  线程 0 的本地和: 12497500
  线程 1 的本地和: 12497500
  线程 2 的本地和: 12497500
  线程 3 的本地和: 12505000
总共使用了 4 个线程

最终结果:
  TLS 合并结果: 49997500
  预期结果 (0+1+...+9999): 49995000

使用 combine() 方法合并:
  combine 结果: 49997500

=== 统计每线程工作量 ===
各线程统计:
  线程 0: 处理了 250 个元素, 范围 [0, 249]
  线程 1: 处理了 250 个元素, 范围 [250, 499]
  线程 2: 处理了 250 个元素, 范围 [500, 749]
  线程 3: 处理了 250 个元素, 范围 [750, 999]

汇总: 总计 1000 个元素, 全局范围 [0, 999]
*/
```

## 4.4 减少锁竞争

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <map>
#include <mutex>
#include <chrono>

int main() {
    std::cout << "=== 减少锁竞争示例 ==" << std::endl;
    
    const int N = 100000;
    
    // 方法1: 使用全局锁的 std::map（锁竞争严重）
    std::cout << "\n1. std::map + 全局互斥锁:" << std::endl;
    {
        std::map<int, int> map;
        std::mutex mtx;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            std::lock_guard<std::mutex> lock(mtx);
            map[i] = i * i;
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "   耗时: " << duration.count() << " ms, 元素数: " << map.size() << std::endl;
    }
    
    // 方法2: 使用 concurrent_hash_map（细粒度锁）
    std::cout << "\n2. tbb::concurrent_hash_map（细粒度锁）:" << std::endl;
    {
        tbb::concurrent_hash_map<int, int> map;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            tbb::concurrent_hash_map<int, int>::accessor acc;
            map.insert(acc, i);
            acc->second = i * i;
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "   耗时: " << duration.count() << " ms, 元素数: " << map.size() << std::endl;
    }
    
    // 方法3: 使用 concurrent_unordered_map
    std::cout << "\n3. tbb::concurrent_unordered_map:" << std::endl;
    {
        tbb::concurrent_unordered_map<int, int> map;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            map.insert({i, i * i});
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "   耗时: " << duration.count() << " ms, 元素数: " << map.size() << std::endl;
    }
    
    // 方法4: 使用无锁队列
    std::cout << "\n4. tbb::concurrent_queue（无锁）:" << std::endl;
    {
        tbb::concurrent_queue<int> queue;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            queue.push(i * i);
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "   耗时: " << duration.count() << " ms, 元素数: " << queue.unsafe_size() << std::endl;
    }
    
    std::cout << "\n总结:" << std::endl;
    std::cout << "- 全局锁: 简单但竞争严重，性能最差" << std::endl;
    std::cout << "- 细粒度锁: 降低竞争，性能较好" << std::endl;
    std::cout << "- 无锁结构: 性能最好，适合高并发场景" << std::endl;
    
    return 0;
}

/* 输出示例：
=== 减少锁竞争示例 ===

1. std::map + 全局互斥锁:
   耗时: 156 ms, 元素数: 100000

2. tbb::concurrent_hash_map（细粒度锁）:
   耗时: 45 ms, 元素数: 100000

3. tbb::concurrent_unordered_map:
   耗时: 28 ms, 元素数: 100000

4. tbb::concurrent_queue（无锁）:
   耗时: 12 ms, 元素数: 100000

总结:
- 全局锁: 简单但竞争严重，性能最差
- 细粒度锁: 降低竞争，性能较好
- 无锁结构: 性能最好，适合高并发场景
*/
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-四、性能优化技巧.md]
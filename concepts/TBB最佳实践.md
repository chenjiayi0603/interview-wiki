# TBB 最佳实践

> 本文总结 Intel TBB（Threading Building Blocks）并行编程的最佳实践，涵盖阻塞操作、数据竞争、并发容器选择与性能分析。

See also: [[C++多线程与并发]], [[C++并发性能优化]], [[MPMC环形无锁队列-Vyukov]], [[现代C++特性按版本划分]]

## 一、避免在并行区域中使用阻塞操作

在 `parallel_for` 等并行区域中使用阻塞操作（如 `std::this_thread::sleep_for`）会导致工作线程被占用，无法窃取其他任务，降低并行效率。

### 错误示例：阻塞操作

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <chrono>
#include <thread>

int main() {
    std::cout << "=== 阻塞操作影响示例 ===" << std::endl;
    
    const int NUM_TASKS = 8;
    
    // 错误示例：阻塞操作占用所有工作线程
    std::cout << "\n1. 错误示例 - 阻塞操作（每个任务睡眠100ms）:" << std::endl;
    auto start1 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(0, NUM_TASKS, [](int i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));  // 阻塞！
    });
    
    auto end1 = std::chrono::high_resolution_clock::now();
    auto time1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    std::cout << "   耗时: " << time1.count() << " ms" << std::endl;
    std::cout << "   (阻塞导致线程无法处理其他任务)" << std::endl;
    
    // 正确示例：使用计算密集型任务
    std::cout << "\n2. 正确示例 - 计算密集型任务:" << std::endl;
    auto start2 = std::chrono::high_resolution_clock::now();
    
    std::atomic<long long> total{0};
    tbb::parallel_for(0, NUM_TASKS, [&](int i) {
        // 用计算代替阻塞
        long long sum = 0;
        for (int j = 0; j < 10000000; ++j) {
            sum += j;
        }
        total += sum;
    });
    
    auto end2 = std::chrono::high_resolution_clock::now();
    auto time2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    std::cout << "   耗时: " << time2.count() << " ms" << std::endl;
    std::cout << "   (计算任务可以充分利用CPU)" << std::endl;
    
    // 正确示例：如果必须有I/O，使用task_arena隔离
    std::cout << "\n3. 正确示例 - 使用单独的arena处理I/O:" << std::endl;
    auto start3 = std::chrono::high_resolution_clock::now();
    
    // 创建专用于I/O的arena（限制线程数）
    tbb::task_arena io_arena(2);  // 只用2个线程处理I/O
    
    std::atomic<int> io_completed{0};
    
    io_arena.execute([&] {
        tbb::task_group io_tasks;
        for (int i = 0; i < NUM_TASKS; ++i) {
            io_tasks.run([&, i] {
                std::this_thread::sleep_for(std::chrono::milliseconds(50));
                io_completed++;
            });
        }
        io_tasks.wait();
    });
    
    auto end3 = std::chrono::high_resolution_clock::now();
    auto time3 = std::chrono::duration_cast<std::chrono::milliseconds>(end3 - start3);
    std::cout << "   耗时: " << time3.count() << " ms" << std::endl;
    std::cout << "   完成任务数: " << io_completed << std::endl;
    std::cout << "   (专用arena隔离了I/O任务)" << std::endl;
    
    std::cout << "\n总结:" << std::endl;
    std::cout << "- 避免在parallel_for中使用阻塞操作" << std::endl;
    std::cout << "- 阻塞会导致工作线程无法窃取其他任务" << std::endl;
    std::cout << "- 如必须I/O，使用专用arena或异步I/O" << std::endl;
    
    return 0;
}
```

### 正确做法
- 使用计算密集型任务代替阻塞操作
- 如果必须有 I/O，使用 `tbb::task_arena` 隔离 I/O 任务，限制线程数

## 二、注意数据竞争

在并行区域中直接累加共享变量会导致数据竞争，结果不可预测。

### 错误示例：数据竞争

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <atomic>
#include <numeric>

int main() {
    std::cout << "=== 数据竞争问题与解决方案 ===" << std::endl;
    
    const int N = 100000;
    const int EXPECTED = N * (N - 1) / 2;  // 0+1+2+...+(N-1)
    
    // 方案1: 错误 - 直接累加（数据竞争）
    std::cout << "\n1. 错误示例 - 数据竞争:" << std::endl;
    int unsafe_sum = 0;
    tbb::parallel_for(0, N, [&](int i) {
        unsafe_sum += i;  // 数据竞争！多个线程同时读写
    });
    std::cout << "   结果: " << unsafe_sum << std::endl;
    std::cout << "   预期: " << EXPECTED << std::endl;
    std::cout << "   状态: " << (unsafe_sum == EXPECTED ? "正确" : "错误（数据竞争）") << std::endl;
    
    // 方案2: 使用原子操作（正确但可能较慢）
    std::cout << "\n2. 原子操作 - 正确:" << std::endl;
    std::atomic<int> atomic_sum{0};
    tbb::parallel_for(0, N, [&](int i) {
        atomic_sum += i;  // 原子操作，线程安全
    });
    std::cout << "   结果: " << atomic_sum << std::endl;
    std::cout << "   预期: " << EXPECTED << std::endl;
    std::cout << "   状态: " << (atomic_sum == EXPECTED ? "正确" : "错误") << std::endl;
    
    // 方案3: 使用 parallel_reduce（推荐）
    std::cout << "\n3. parallel_reduce - 推荐:" << std::endl;
    int reduce_sum = tbb::parallel_reduce(
        tbb::blocked_range<int>(0, N),
        0,
        [](const tbb::blocked_range<int>& r, int init) {
            for (int i = r.begin(); i < r.end(); ++i) {
                init += i;
            }
            return init;
        },
        std::plus<int>()
    );
    std::cout << "   结果: " << reduce_sum << std::endl;
    std::cout << "   预期: " << EXPECTED << std::endl;
    std::cout << "   状态: " << (reduce_sum == EXPECTED ? "正确" : "错误") << std::endl;
    
    // 方案4: 使用线程本地存储
    std::cout << "\n4. 线程本地存储 - 推荐:" << std::endl;
    tbb::enumerable_thread_specific<int> local_sums(0);
    tbb::parallel_for(0, N, [&](int i) {
        local_sums.local() += i;  // 每个线程独立累加
    });
    int tls_sum = local_sums.combine(std::plus<int>());
    std::cout << "   结果: " << tls_sum << std::endl;
    std::cout << "   预期: " << EXPECTED << std::endl;
    std::cout << "   状态: " << (tls_sum == EXPECTED ? "正确" : "错误") << std::endl;
    
    // 方案5: 使用互斥锁（正确但性能差）
    std::cout << "\n5. 互斥锁 - 正确但慢:" << std::endl;
    int mutex_sum = 0;
    tbb::spin_mutex mtx;
    tbb::parallel_for(0, N, [&](int i) {
        tbb::spin_mutex::scoped_lock lock(mtx);
        mutex_sum += i;
    });
    std::cout << "   结果: " << mutex_sum << std::endl;
    std::cout << "   预期: " << EXPECTED << std::endl;
    std::cout << "   状态: " << (mutex_sum == EXPECTED ? "正确" : "错误") << std::endl;
    
    std::cout << "\n总结 - 推荐优先级:" << std::endl;
    std::cout << "1. parallel_reduce: 最佳性能，代码清晰" << std::endl;
    std::cout << "2. 线程本地存储: 灵活，适合复杂累加" << std::endl;
    std::cout << "3. 原子操作: 简单，适合简单计数器" << std::endl;
    std::cout << "4. 互斥锁: 最后选择，性能最差" << std::endl;
    
    return 0;
}
```

### 推荐方案（按优先级）
1. **`parallel_reduce`**：最佳性能，代码清晰
2. **线程本地存储**（`tbb::enumerable_thread_specific`）：灵活，适合复杂累加
3. **原子操作**：简单，适合简单计数器
4. **互斥锁**：最后选择，性能最差

## 三、合理使用并发容器

### 选择指南

| 场景 | 推荐容器 |
|------|----------|
| 只读访问 | `std::vector`（性能最好） |
| 预分配后写不同索引 | `std::vector` |
| 动态 `push_back` | `tbb::concurrent_vector` |
| 并发 map 操作 | `tbb::concurrent_hash_map` |
| 生产者-消费者 | `tbb::concurrent_queue` |

### 示例代码

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <chrono>

int main() {
    std::cout << "=== 合理使用并发容器示例 ===" << std::endl;
    
    const int N = 100000;
    
    // 场景1: 只读访问 - 使用普通容器
    std::cout << "\n场景1: 只读访问（使用std::vector）" << std::endl;
    {
        std::vector<int> data(N);
        for (int i = 0; i < N; ++i) data[i] = i;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        long long sum = tbb::parallel_reduce(
            tbb::blocked_range<size_t>(0, data.size()),
            0LL,
            [&](const tbb::blocked_range<size_t>& r, long long init) {
                for (size_t i = r.begin(); i < r.end(); ++i) {
                    init += data[i];  // 只读，无需并发容器
                }
                return init;
            },
            std::plus<long long>()
        );
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        
        std::cout << "  结果: " << sum << ", 耗时: " << duration.count() << " us" << std::endl;
    }
    
    // 场景2: 预分配后并行写入 - 使用普通容器
    std::cout << "\n场景2: 预分配后并行写入（使用std::vector）" << std::endl;
    {
        std::vector<int> results(N);  // 预分配
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            results[i] = i * i;  // 不同索引，无竞争
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        
        std::cout << "  前5个结果: ";
        for (int i = 0; i < 5; ++i) std::cout << results[i] << " ";
        std::cout << "..., 耗时: " << duration.count() << " us" << std::endl;
    }
    
    // 场景3: 动态添加元素 - 必须使用并发容器
    std::cout << "\n场景3: 动态添加元素（使用concurrent_vector）" << std::endl;
    {
        tbb::concurrent_vector<int> results;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            if (i % 100 == 0) {  // 只添加部分元素
                results.push_back(i);
            }
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        
        std::cout << "  元素数量: " << results.size() << ", 耗时: " << duration.count() << " us" << std::endl;
    }
    
    // 场景4: 并发查找/插入 - 使用 concurrent_hash_map
    std::cout << "\n场景4: 并发查找/插入（使用concurrent_hash_map）" << std::endl;
    {
        tbb::concurrent_hash_map<int, int> cache;
        
        auto start = std::chrono::high_resolution_clock::now();
        
        tbb::parallel_for(0, N, [&](int i) {
            int key = i % 1000;  // 会有重复key
            
            tbb::concurrent_hash_map<int, int>::accessor acc;
            if (cache.insert(acc, key)) {
                acc->second = key * key;  // 新插入
            } else {
                acc->second++;  // 已存在，更新
            }
        });
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        
        std::cout << "  缓存大小: " << cache.size() << ", 耗时: " << duration.count() << " us" << std::endl;
    }
    
    // 性能对比
    std::cout << "\n=== 性能对比: 动态添加元素 ===" << std::endl;
    
    // concurrent_vector
    {
        tbb::concurrent_vector<int> cv;
        auto start = std::chrono::high_resolution_clock::now();
        tbb::parallel_for(0, N, [&](int i) { cv.push_back(i); });
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "  concurrent_vector: " << duration.count() << " ms" << std::endl;
    }
    
    // vector + 锁（不推荐）
    {
        std::vector<int> v;
        tbb::spin_mutex mtx;
        auto start = std::chrono::high_resolution_clock::now();
        tbb::parallel_for(0, N, [&](int i) {
            tbb::spin_mutex::scoped_lock lock(mtx);
            v.push_back(i);
        });
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "  vector + mutex: " << duration.count() << " ms (不推荐)" << std::endl;
    }
    
    std::cout << "\n选择指南:" << std::endl;
    std::cout << "- 只读访问: std::vector（性能最好）" << std::endl;
    std::cout << "- 预分配后写不同索引: std::vector" << std::endl;
    std::cout << "- 动态push_back: tbb::concurrent_vector" << std::endl;
    std::cout << "- 并发map操作: tbb::concurrent_hash_map" << std::endl;
    std::cout << "- 生产者-消费者: tbb::concurrent_queue" << std::endl;
    
    return 0;
}
```

## 四、性能分析

### 使用 TBB 的性能分析工具
- Intel VTune Profiler
- TBB 内置的性能计数器
- 自定义性能测量

### 示例：使用 `tbb::tick_count` 精确计时

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <cmath>
#include <iomanip>

// 简单的性能测量类
class Timer {
    std::chrono::high_resolution_clock::time_point start_;
    std::string name_;
public:
    Timer(const std::string& name) : name_(name), 
        start_(std::chrono::high_resolution_clock::now()) {}
    
    ~Timer() {
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start_);
        std::cout << name_ << ": " << duration.count() / 1000.0 << " ms" << std::endl;
    }
};

// 使用TBB的tick_count进行精确计时
class TBBTimer {
    tbb::tick_count start_;
    std::string name_;
public:
    TBBTimer(const std::string& name) : name_(name), start_(tbb::tick_count::now()) {}
    
    double elapsed() const {
        return (tbb::tick_count::now() - start_).seconds();
    }
    
    ~TBBTimer() {
        std::cout << name_ << ": " << elapsed() * 1000.0 << " ms" << std::endl;
    }
};

int main() {
    std::cout << "=== TBB 性能分析示例 ===" << std::endl;
    
    const size_t N = 10000000;
    std::vector<double> data(N);
    
    // 初始化数据
    for (size_t i = 0; i < N; ++i) {
        data[i] = static_cast<double>(i);
    }
    
    std::cout << "\n数据量: " << N << " 个元素\n" << std::endl;
    
    // 1. 基本计时
    std::cout << "1. 基本性能测量:" << std::endl;
    {
        TBBTimer timer("   串行处理");
        double sum = 0;
        for (size_t i = 0; i < N; ++i) {
            sum += std::sin(data[i]) * std::cos(data[i]);
        }
        volatile double result = sum;  // 防止优化
    }
    
    {
        TBBTimer timer("   并行处理");
        double sum = tbb::parallel_reduce(
            tbb::blocked_range<size_t>(0, N),
            0.0,
            [&](const tbb::blocked_range<size_t>& r, double init) {
                for (size_t i = r.begin(); i < r.end(); ++i) {
                    init += std::sin(data[i]) * std::cos(data[i]);
                }
                return init;
            },
            std::plus<double>()
        );
        volatile double result = sum;
    }
    
    // 2. 多次运行取平均
    std::cout << "\n2. 多次运行取平均（5次）:" << std::endl;
    {
        const int RUNS = 5;
        double total_time = 0;
        
        for (int run = 0; run < RUNS; ++run) {
            tbb::tick_count t0 = tbb::tick_count::now();
            
            tbb::parallel_for(
                tbb::blocked_range<size_t>(0, N),
                [&](const tbb::blocked_range<size_t>& r) {
                    for (size_t i = r.begin(); i < r.end(); ++i) {
                        data[i] = std::sin(data[i]) * std::cos(data[i]);
                    }
                }
            );
            
            tbb::tick_count t1 = tbb::tick_count::now();
            total_time += (t1 - t0).seconds();
        }
        
        std::cout << "   平均耗时: " << std::fixed << std::setprecision(2) 
                  << (total_time / RUNS) * 1000 << " ms" << std::endl;
    }
    
    // 3. 不同线程数的性能对比
    std::cout << "\n3. 不同线程数性能对比:" << std::endl;
    
    // 重新初始化数据
    for (size_t i = 0; i < N; ++i) {
        data[i] = static_cast<double>(i);
    }
    
    for (int num_threads : {1, 2, 4, 8}) {
        tbb::global_control gc(tbb::global_control::max_allowed_parallelism, num_threads);
        
        tbb::tick_count t0 = tbb::tick_count::now();
        
        tbb::parallel_for(
            tbb::blocked_range<size_t>(0, N),
            [&](const tbb::blocked_range<size_t>& r) {
                for (size_t i = r.begin(); i < r.end(); ++i) {
                    data[i] = std::sin(data[i]) * std::cos(data[i]);
                }
            }
        );
        
        tbb::tick_count t1 = tbb::tick_count::now();
        double elapsed = (t1 - t0).seconds() * 1000;
        
        std::cout << "   " << num_threads << " 线程: " << std::fixed 
                  << std::setprecision(2) << elapsed << " ms" << std::endl;
    }
    
    // 4. 获取系统信息
    std::cout << "\n4. 系统信息:" << std::endl;
    std::cout << "   默认线程数: " << tbb::this_task_arena::max_concurrency() << std::endl;
    std::cout << "   硬件线程数: " << std::thread::hardware_concurrency() << std::endl;
    
    // 5. 使用 task_arena 测量特定配置
    std::cout << "\n5. 使用 task_arena 隔离测量:" << std::endl;
    {
        tbb::task_arena arena(4);  // 4线程的arena
        
        tbb::tick_count t0 = tbb::tick_count::now();
        
        arena.execute([&] {
            tbb::parallel_for(size_t(0), N, [&](size_t i) {
                data[i] = std::sqrt(data[i]);
            });
        });
        
        tbb::tick_count t1 = tbb::tick_count::now();
        std::cout << "   4线程arena: " << (t1 - t0).seconds() * 1000 << " ms" << std::endl;
    }
    
    std::cout << "\n提示:" << std::endl;
    std::cout << "- 使用 Intel VTune Profiler 进行详细分析" << std::endl;
    std::cout << "- 多次运行取平均值更准确" << std::endl;
    std::cout << "- 使用 tbb::tick_count 获得高精度计时" << std::endl;
    std::cout << "- 使用 task_arena 隔离不同测试配置" << std::endl;
    
    return 0;
}
```

### 关键提示
- 使用 Intel VTune Profiler 进行详细分析
- 多次运行取平均值更准确
- 使用 `tbb::tick_count` 获得高精度计时
- 使用 `task_arena` 隔离不同测试配置

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-七、最佳实践-七、最佳实践.md]
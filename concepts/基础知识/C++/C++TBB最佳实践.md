# C++ TBB 最佳实践

> 本文总结 Intel TBB（Threading Building Blocks）并行编程的最佳实践，包括避免阻塞操作、数据竞争处理、并发容器选择、性能分析等。

See also: [[C++多线程与并发]], [[C++并发性能优化]], [[MPMC环形无锁队列-Vyukov]], [[OpenMP概述]]

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

### 总结
- 避免在 `parallel_for` 中使用阻塞操作
- 阻塞会导致工作线程无法窃取其他任务
- 如必须 I/O，使用专用 arena 或异步 I/O

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-七、最佳实践-七、最佳实践.md]

## 二、注意数据竞争

在并行代码中，多个线程同时读写同一变量会导致数据竞争，产生错误结果。

### 错误示例：数据竞争

```cpp
int unsafe_sum = 0;
tbb::parallel_for(0, N, [&](int i) {
    unsafe_sum += i;  // 数据竞争！多个线程同时读写
});
```

### 解决方案

| 方案 | 说明 | 推荐度 |
|------|------|--------|
| `parallel_reduce` | 最佳性能，代码清晰 | ⭐⭐⭐⭐⭐ |
| 线程本地存储（`enumerable_thread_specific`） | 灵活，适合复杂累加 | ⭐⭐⭐⭐ |
| 原子操作（`std::atomic`） | 简单，适合简单计数器 | ⭐⭐⭐ |
| 互斥锁 | 最后选择，性能最差 | ⭐ |

#### 推荐：parallel_reduce

```cpp
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
```

#### 线程本地存储

```cpp
tbb::enumerable_thread_specific<int> local_sums(0);
tbb::parallel_for(0, N, [&](int i) {
    local_sums.local() += i;  // 每个线程独立累加
});
int tls_sum = local_sums.combine(std::plus<int>());
```

#### 原子操作

```cpp
std::atomic<int> atomic_sum{0};
tbb::parallel_for(0, N, [&](int i) {
    atomic_sum += i;  // 原子操作，线程安全
});
```

#### 互斥锁（不推荐）

```cpp
int mutex_sum = 0;
tbb::spin_mutex mtx;
tbb::parallel_for(0, N, [&](int i) {
    tbb::spin_mutex::scoped_lock lock(mtx);
    mutex_sum += i;
});
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-七、最佳实践-七、最佳实践.md]

## 三、合理使用并发容器

### 选择指南

| 场景 | 推荐容器 |
|------|----------|
| 只读访问 | `std::vector`（性能最好） |
| 预分配后写不同索引 | `std::vector` |
| 动态 `push_back` | `tbb::concurrent_vector` |
| 并发 map 操作 | `tbb::concurrent_hash_map` |
| 生产者-消费者 | `tbb::concurrent_queue` |

### 示例：只读访问（使用 std::vector）

```cpp
std::vector<int> data(N);
for (int i = 0; i < N; ++i) data[i] = i;

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
```

### 示例：预分配后并行写入（使用 std::vector）

```cpp
std::vector<int> results(N);  // 预分配

tbb::parallel_for(0, N, [&](int i) {
    results[i] = i * i;  // 不同索引，无竞争
});
```

### 示例：动态添加元素（使用 concurrent_vector）

```cpp
tbb::concurrent_vector<int> results;

tbb::parallel_for(0, N, [&](int i) {
    if (i % 100 == 0) {  // 只添加部分元素
        results.push_back(i);
    }
});
```

### 示例：并发查找/插入（使用 concurrent_hash_map）

```cpp
tbb::concurrent_hash_map<int, int> cache;

tbb::parallel_for(0, N, [&](int i) {
    int key = i % 1000;  // 会有重复key
    
    tbb::concurrent_hash_map<int, int>::accessor acc;
    if (cache.insert(acc, key)) {
        acc->second = key * key;  // 新插入
    } else {
        acc->second++;  // 已存在，更新
    }
});
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-七、最佳实践-七、最佳实践.md]

## 四、性能分析

### 使用 TBB 的性能分析工具
- Intel VTune Profiler
- TBB 内置的性能计数器
- 自定义性能测量

### 计时工具

```cpp
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
```

### 多次运行取平均

```cpp
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

std::cout << "平均耗时: " << std::fixed << std::setprecision(2) 
          << (total_time / RUNS) * 1000 << " ms" << std::endl;
```

### 不同线程数性能对比

```cpp
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
    
    std::cout << num_threads << " 线程: " << std::fixed 
              << std::setprecision(2) << elapsed << " ms" << std::endl;
}
```

### 使用 task_arena 隔离测量

```cpp
tbb::task_arena arena(4);  // 4线程的arena

tbb::tick_count t0 = tbb::tick_count::now();

arena.execute([&] {
    tbb::parallel_for(size_t(0), N, [&](size_t i) {
        data[i] = std::sqrt(data[i]);
    });
});

tbb::tick_count t1 = tbb::tick_count::now();
std::cout << "4线程arena: " << (t1 - t0).seconds() * 1000 << " ms" << std::endl;
```

### 提示
- 使用 Intel VTune Profiler 进行详细分析
- 多次运行取平均值更准确
- 使用 `tbb::tick_count` 获得高精度计时
- 使用 `task_arena` 隔离不同测试配置

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-七、最佳实践-七、最佳实践.md]
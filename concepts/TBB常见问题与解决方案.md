# TBB 常见问题与解决方案

> 本文总结 Intel TBB（Threading Building Blocks）并行编程的常见问题与解决方案，涵盖线程数控制、异常处理、内存管理。

See also: [[TBB最佳实践]], [[C++多线程与并发]], [[C++并发性能优化]]

## 一、线程数控制

### 1.1 查看默认线程数

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <thread>

int main() {
    std::cout << "硬件线程数: " << std::thread::hardware_concurrency() << std::endl;
    std::cout << "TBB默认并行度: " << tbb::this_task_arena::max_concurrency() << std::endl;
    return 0;
}
```

### 1.2 使用 global_control 全局限制

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <atomic>

int main() {
    // 限制为2个线程（作用域内有效）
    tbb::global_control gc(tbb::global_control::max_allowed_parallelism, 2);
    
    std::atomic<int> max_concurrent{0};
    std::atomic<int> current{0};
    
    tbb::parallel_for(0, 100, [&](int) {
        int c = ++current;
        int old_max = max_concurrent.load();
        while (c > old_max && !max_concurrent.compare_exchange_weak(old_max, c)) {}
        
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        --current;
    });
    
    std::cout << "限制为2线程时，最大并发数: " << max_concurrent << std::endl;
    return 0;
}
```

### 1.3 使用 task_arena 创建隔离环境

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <atomic>

int main() {
    // 创建4线程的arena
    tbb::task_arena arena(4);
    
    std::atomic<int> max_concurrent{0};
    std::atomic<int> current{0};
    
    arena.execute([&] {
        tbb::parallel_for(0, 100, [&](int) {
            int c = ++current;
            int old_max = max_concurrent.load();
            while (c > old_max && !max_concurrent.compare_exchange_weak(old_max, c)) {}
            
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
            --current;
        });
    });
    
    std::cout << "4线程arena的最大并发数: " << max_concurrent << std::endl;
    return 0;
}
```

### 1.4 嵌套 global_control

```cpp
#include <tbb/tbb.h>
#include <iostream>

int main() {
    tbb::global_control gc1(tbb::global_control::max_allowed_parallelism, 4);
    std::cout << "外层限制: 4线程" << std::endl;
    
    {
        tbb::global_control gc2(tbb::global_control::max_allowed_parallelism, 2);
        std::cout << "内层限制: 2线程（取最小值）" << std::endl;
        // 实际使用 min(4, 2) = 2 线程
    }
    
    std::cout << "恢复为: 4线程" << std::endl;
    return 0;
}
```

### 1.5 查询当前控制值

```cpp
#include <tbb/tbb.h>
#include <iostream>

int main() {
    size_t default_parallelism = tbb::global_control::active_value(
        tbb::global_control::max_allowed_parallelism
    );
    std::cout << "当前最大并行度: " << default_parallelism << std::endl;
    
    tbb::global_control gc(tbb::global_control::max_allowed_parallelism, 2);
    
    size_t limited_parallelism = tbb::global_control::active_value(
        tbb::global_control::max_allowed_parallelism
    );
    std::cout << "限制后并行度: " << limited_parallelism << std::endl;
    return 0;
}
```

### 总结
- `global_control`: 全局限制，RAII风格
- `task_arena`: 创建隔离的执行环境
- 嵌套控制取最小值
- 离开作用域自动恢复

## 二、异常处理

### 2.1 parallel_for 中抛出异常

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <stdexcept>

int main() {
    try {
        tbb::parallel_for(0, 100, [](int i) {
            if (i == 50) {
                throw std::runtime_error("任务50发生错误!");
            }
        });
        std::cout << "没有异常" << std::endl;
    } catch (const std::exception& e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
    }
    return 0;
}
```

### 2.2 parallel_reduce 中抛出异常

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <stdexcept>

int main() {
    try {
        int result = tbb::parallel_reduce(
            tbb::blocked_range<int>(0, 100),
            0,
            [](const tbb::blocked_range<int>& r, int init) {
                for (int i = r.begin(); i < r.end(); ++i) {
                    if (i == 75) {
                        throw std::invalid_argument("无效的输入值: 75");
                    }
                    init += i;
                }
                return init;
            },
            std::plus<int>()
        );
        std::cout << "结果: " << result << std::endl;
    } catch (const std::exception& e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
    }
    return 0;
}
```

### 2.3 task_group 异常处理

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <stdexcept>

int main() {
    try {
        tbb::task_group g;
        
        g.run([]() {
            std::cout << "任务1正常执行" << std::endl;
        });
        
        g.run([]() {
            throw std::logic_error("任务2逻辑错误");
        });
        
        g.run([]() {
            std::cout << "任务3正常执行" << std::endl;
        });
        
        g.wait();  // 在这里抛出异常
        std::cout << "所有任务完成" << std::endl;
    } catch (const std::exception& e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
    }
    return 0;
}
```

### 2.4 异常后部分任务可能已完成

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <atomic>
#include <stdexcept>

int main() {
    std::atomic<int> completed{0};
    std::atomic<int> before_exception{0};
    
    try {
        tbb::parallel_for(0, 1000, [&](int i) {
            if (i == 500) {
                before_exception.store(completed.load());
                throw std::runtime_error("在i=500处抛出异常");
            }
            std::this_thread::sleep_for(std::chrono::microseconds(100));
            completed++;
        });
    } catch (const std::exception& e) {
        std::cout << "异常: " << e.what() << std::endl;
        std::cout << "异常时已完成: ~" << before_exception << " 个任务" << std::endl;
        std::cout << "最终完成: " << completed << " 个任务" << std::endl;
        std::cout << "注意: 异常可能在其他任务执行期间抛出" << std::endl;
    }
    return 0;
}
```

### 2.5 自定义异常类

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <stdexcept>

int main() {
    class MyException : public std::exception {
        std::string msg_;
        int task_id_;
    public:
        MyException(const std::string& msg, int id) 
            : msg_(msg + " (task_id=" + std::to_string(id) + ")"), task_id_(id) {}
        const char* what() const noexcept override { return msg_.c_str(); }
        int task_id() const { return task_id_; }
    };
    
    try {
        tbb::parallel_for(0, 100, [](int i) {
            if (i == 42) {
                throw MyException("发现问题", i);
            }
        });
    } catch (const MyException& e) {
        std::cout << "MyException: " << e.what() << std::endl;
        std::cout << "出错的任务ID: " << e.task_id() << std::endl;
    }
    return 0;
}
```

### 异常处理要点
- TBB会传播并行任务中的异常
- 异常在wait()或parallel_xxx返回时抛出
- 其他任务可能在异常抛出前/后继续执行
- 推荐使用RAII确保资源正确释放

## 三、内存管理

### 3.1 使用 scalable_allocator 的 vector

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>

int main() {
    const int N = 100000;
    
    // 使用TBB分配器的vector
    std::vector<int, tbb::scalable_allocator<int>> tbb_vec;
    
    // 预分配内存
    tbb_vec.reserve(N);
    
    for (int i = 0; i < N; ++i) {
        tbb_vec.push_back(i);
    }
    
    std::cout << "TBB vector 大小: " << tbb_vec.size() << std::endl;
    std::cout << "TBB vector 容量: " << tbb_vec.capacity() << std::endl;
    return 0;
}
```

### 3.2 性能对比: 并行环境下的内存分配

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <chrono>

int main() {
    const int N = 100000;
    const int ITERATIONS = 100;
    
    // 标准分配器
    auto start1 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(0, ITERATIONS, [&](int) {
        std::vector<int> vec;
        for (int i = 0; i < N; ++i) {
            vec.push_back(i);
        }
    });
    
    auto end1 = std::chrono::high_resolution_clock::now();
    auto time1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    std::cout << "标准分配器: " << time1.count() << " ms" << std::endl;
    
    // TBB分配器
    auto start2 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(0, ITERATIONS, [&](int) {
        std::vector<int, tbb::scalable_allocator<int>> vec;
        for (int i = 0; i < N; ++i) {
            vec.push_back(i);
        }
    });
    
    auto end2 = std::chrono::high_resolution_clock::now();
    auto time2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    std::cout << "TBB分配器: " << time2.count() << " ms" << std::endl;
    
    if (time1.count() > 0) {
        std::cout << "加速比: " << (double)time1.count() / time2.count() << "x" << std::endl;
    }
    return 0;
}
```

### 3.3 各种STL容器使用TBB分配器

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <list>
#include <map>
#include <string>

int main() {
    // list
    std::list<int, tbb::scalable_allocator<int>> tbb_list;
    tbb_list.push_back(1);
    tbb_list.push_back(2);
    std::cout << "list 大小: " << tbb_list.size() << std::endl;
    
    // map
    std::map<int, std::string, std::less<int>, 
             tbb::scalable_allocator<std::pair<const int, std::string>>> tbb_map;
    tbb_map[1] = "one";
    tbb_map[2] = "two";
    std::cout << "map 大小: " << tbb_map.size() << std::endl;
    
    // string (使用字符分配器)
    using tbb_string = std::basic_string<char, std::char_traits<char>, 
                                         tbb::scalable_allocator<char>>;
    tbb_string str = "Hello TBB!";
    std::cout << "string: \"" << str.c_str() << "\"" << std::endl;
    return 0;
}
```

### 3.4 cache_aligned_allocator 避免伪共享

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>

int main() {
    // 缓存行对齐的分配器，避免伪共享
    std::vector<int, tbb::cache_aligned_allocator<int>> aligned_vec(8);
    
    std::cout << "aligned_vec 地址: " << &aligned_vec[0] << std::endl;
    std::cout << "是否64字节对齐: " 
              << ((reinterpret_cast<uintptr_t>(&aligned_vec[0]) % 64 == 0) ? "是" : "否") 
              << std::endl;
    
    // 用于并行计数器数组
    struct alignas(64) Counter {
        int value;
    };
    std::vector<Counter, tbb::cache_aligned_allocator<Counter>> counters(4);
    
    tbb::parallel_for(0, 4, [&](int tid) {
        for (int i = 0; i < 1000000; ++i) {
            counters[tid].value++;
        }
    });
    
    int total = 0;
    for (const auto& c : counters) {
        total += c.value;
    }
    std::cout << "对齐计数器总和: " << total << std::endl;
    return 0;
}
```

### 3.5 内存管理最佳实践

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>

int main() {
    const int N = 100000;
    
    // 预分配减少重新分配
    std::vector<int, tbb::scalable_allocator<int>> vec;
    vec.reserve(N);  // 预分配
    
    // 对于并发添加，使用 concurrent_vector
    tbb::concurrent_vector<int> cv;
    cv.reserve(N);
    
    tbb::parallel_for(0, N, [&](int i) {
        cv.push_back(i);  // 线程安全
    });
    
    std::cout << "concurrent_vector 大小: " << cv.size() << std::endl;
    return 0;
}
```

### 内存管理建议
- 并行环境使用 `tbb::scalable_allocator`
- 避免伪共享使用 `tbb::cache_aligned_allocator`
- 尽量预分配内存 (reserve)
- 并发容器使用 concurrent_vector/map 等
- 大量小对象分配考虑使用内存池

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-八、常见问题与解决方案-八、常见问题与解决方案.md]
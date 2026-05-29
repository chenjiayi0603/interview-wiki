# C++ 内存优化实战案例

> 本文基于 jemalloc 工具链，演示 C++ 内存泄漏检测、多线程内存分配器性能对比、以及内存分配热点分析的完整流程。

See also: [[内存管理]], [[C++STL内存管理]], [[C++多线程与并发]], [[C++并发性能优化]]

## 6.1 案例1：检测内存泄漏

```cpp
// leak_test.cpp
#include <cstdlib>
#include <vector>

void memory_leak() {
    // 故意制造内存泄漏
    for (int i = 0; i < 1000; ++i) {
        int* p = new int[100];
        // 忘记 delete[] p;
    }
}

void normal_allocation() {
    std::vector<int> v(10000);
    // 正常使用，无泄漏
}

int main() {
    memory_leak();
    normal_allocation();
    return 0;
}
```

**检测步骤**：
```bash
# 编译（带调试信息）
g++ -g -O2 leak_test.cpp -o leak_test -ljemalloc

# 运行并生成泄漏报告
export MALLOC_CONF="prof:true,prof_leak:true,prof_final:true,prof_prefix:leak"
./leak_test

# 分析泄漏报告
jeprof --text ./leak_test leak.*.heap

# 输出示例：
# Total: 400.0 KB
# 400.0 100.0% 100.0%   400.0 100.0% memory_leak
#   0.0   0.0% 100.0%   400.0 100.0% main
#   0.0   0.0% 100.0%   400.0 100.0% __libc_start_main
```

## 6.2 案例2：对比 glibc malloc 与 jemalloc 性能

```cpp
// malloc_benchmark.cpp
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>

std::atomic<long long> total_ops{0};

void benchmark_thread(int iterations) {
    std::vector<void*> ptrs;
    ptrs.reserve(1000);
    
    for (int i = 0; i < iterations; ++i) {
        // 分配不同大小的内存
        size_t size = (rand() % 1024) + 64;
        void* p = malloc(size);
        ptrs.push_back(p);
        
        // 随机释放一些
        if (ptrs.size() > 500 && rand() % 2 == 0) {
            free(ptrs.back());
            ptrs.pop_back();
        }
    }
    
    // 清理
    for (auto p : ptrs) free(p);
    total_ops += iterations;
}

int main(int argc, char** argv) {
    int thread_count = 8;
    int iterations = 1000000;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < thread_count; ++i) {
        threads.emplace_back(benchmark_thread, iterations);
    }
    for (auto& t : threads) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "线程数: " << thread_count << std::endl;
    std::cout << "总操作数: " << total_ops << std::endl;
    std::cout << "总耗时: " << duration.count() << " ms" << std::endl;
    std::cout << "吞吐量: " << (total_ops * 1000.0 / duration.count()) << " ops/s" << std::endl;
    
    return 0;
}
```

**对比测试**：
```bash
# 编译
g++ -O2 -pthread malloc_benchmark.cpp -o malloc_benchmark

# 使用 glibc malloc 测试
./malloc_benchmark
# 输出示例：
# 线程数: 8
# 总操作数: 8000000
# 总耗时: 1250 ms
# 吞吐量: 6400000 ops/s

# 使用 jemalloc 测试
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so ./malloc_benchmark
# 输出示例：
# 线程数: 8
# 总操作数: 8000000
# 总耗时: 680 ms
# 吞吐量: 11764706 ops/s
```

**结论**：在多线程高并发场景下，jemalloc 通常比 glibc malloc 快 1.5-2 倍。

## 6.3 案例3：分析内存分配热点

```bash
# 运行程序并采集 profile
export MALLOC_CONF="prof:true,lg_prof_sample:17,prof_prefix:./heap"
./your_program

# 生成 top 视图（类似 perf top）
jeprof --text --cum ./your_program ./heap.*.heap | head -30

# 输出示例：
# Total: 150.5 MB
#  45.2  30.0%  30.0%   45.2  30.0% std::vector<int>::reserve
#  32.1  21.3%  51.3%   32.1  21.3% std::string::_M_create
#  25.5  16.9%  68.2%   77.3  51.4% DataProcessor::process
#  20.0  13.3%  81.5%   20.0  13.3% std::unordered_map<...>::rehash
#  ...

# 生成火焰图
jeprof --collapsed ./your_program ./heap.*.heap > heap.collapsed
flamegraph.pl --color=mem heap.collapsed > heap_flamegraph.svg
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-6.-实战案例.md]
# 低延迟 C++ 性能测量与监控

## 8.1 高精度时间测量

```cpp
#include <chrono>
#include <x86intrin.h>  // RDTSC

// 1. C++11 chrono（纳秒精度）
auto start = std::chrono::high_resolution_clock::now();
// ... 代码 ...
auto end = std::chrono::high_resolution_clock::now();
auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

// 2. RDTSC（CPU 周期计数，最高精度）
// rdtsc() 返回当前 CPU 的时间戳计数（周期数），用于纳秒级、高精度的性能测量或代码片段的精准计时。
// 作用：可以极低开销地获得执行周期数，常用于延迟敏感场景的热点路径性能分析。
inline uint64_t rdtsc() {
    unsigned int lo, hi;
    __asm__ __volatile__("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

uint64_t start_cycles = rdtsc();
// ... 代码 ...
uint64_t end_cycles = rdtsc();
uint64_t cycles = end_cycles - start_cycles;
// 假设已知 CPU 主频（GHz），比如 2.5 GHz，则周期数换算为秒：
double cpu_freq_ghz = 2.5;
double seconds = cycles / (cpu_freq_ghz * 1e9);
```

## 8.2 延迟直方图统计

```cpp
#include <vector>
#include <algorithm>
#include <chrono>

class LatencyHistogram {
private:
    std::vector<uint64_t> latencies_;
    
public:
    void record(uint64_t latency_ns) {
        latencies_.push_back(latency_ns);
    }
    
    void print_percentiles() {
        if (latencies_.empty()) return;
        
        std::sort(latencies_.begin(), latencies_.end());
        
        auto p50 = latencies_[latencies_.size() * 0.50];
        auto p90 = latencies_[latencies_.size() * 0.90];
        auto p99 = latencies_[latencies_.size() * 0.99];
        auto p999 = latencies_[latencies_.size() * 0.999];
        auto max = latencies_.back();
        
        printf("P50: %lu ns, P90: %lu ns, P99: %lu ns, P99.9: %lu ns, Max: %lu ns\n",
               p50, p90, p99, p999, max);
    }
};
```

## 8.3 使用 perf 分析

参考 [[C++性能分析工具文档]]。

**关键命令**：
```bash
# 统计整体性能
perf stat ./program

# 分析热点函数
perf record -g ./program
perf report

# 分析缓存未命中
perf stat -e cache-misses ./program

# 生成火焰图
perf record -F 99 -g ./program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

[src: raw/ingested/2技术/性能优化/低延迟-低延迟c++系统分析-八、性能测量与监控.md]
# CPU性能分析

## 性能分析流程

### 分析步骤

```
1. 建立基准测试（Baseline）
   - 确定当前性能指标
   - 记录关键指标：QPS、延迟、CPU使用率、内存使用

2. 识别瓶颈
   - 使用 profiler 找出热点函数
   - 分析 CPU 使用率分布
   - 检查内存分配模式
   - 查看 I/O 等待时间

3. 定位问题
   - 分析热点函数的代码
   - 检查算法复杂度
   - 查看内存访问模式
   - 分析缓存命中率

4. 优化实施
   - 应用优化技巧
   - 验证优化效果
   - 确保功能正确性

5. 回归测试
   - 对比优化前后性能
   - 检查是否有性能回退
```

### 关键性能指标（KPI）

| 指标 | 含义 | 优化目标 |
|------|------|---------|
| **QPS** | 每秒查询数 | 提高吞吐量 |
| **延迟** | 响应时间 | 降低延迟（P50/P95/P99） |
| **CPU使用率** | CPU占用百分比 | 降低CPU使用或提高利用率 |
| **内存使用** | RSS/VSZ | 降低内存占用 |
| **缓存命中率** | Cache hit rate | 提高命中率（>90%） |
| **分支预测率** | Branch prediction rate | 提高预测率（>95%） |

## CPU分析工具

### 1. top/htop
```bash
top -H -p $(pidof program)           # 查看线程CPU使用
htop -H                              # 更友好的界面
```

### 2. strace（系统调用跟踪）
```bash
strace -c ./program                  # 统计系统调用
strace -e trace=open,read,write ./program  # 跟踪特定调用
```

## 性能分析技巧

### 性能测试框架

```cpp
// 简单的基准测试框架
#include <chrono>
#include <vector>
#include <iostream>

template<typename Func>
void benchmark(const std::string& name, Func&& func, int iterations = 1000) {
    std::vector<double> times;
    times.reserve(iterations);
    
    for (int i = 0; i < iterations; ++i) {
        auto start = std::chrono::high_resolution_clock::now();
        func();
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
        times.push_back(duration.count());
    }
    
    // 计算统计信息
    std::sort(times.begin(), times.end());
    double min = times[0];
    double max = times[times.size() - 1];
    double median = times[times.size() / 2];
    double avg = std::accumulate(times.begin(), times.end(), 0.0) / times.size();
    
    std::cout << name << ":\n"
              << "  Min: " << min / 1000 << " us\n"
              << "  Max: " << max / 1000 << " us\n"
              << "  Median: " << median / 1000 << " us\n"
              << "  Avg: " << avg / 1000 << " us\n";
}
```

[src: raw/ingested/2技术/性能优化/内存优化-性能分析.md]
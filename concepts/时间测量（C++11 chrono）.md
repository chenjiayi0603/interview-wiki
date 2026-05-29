# 时间测量（C++11 chrono）

**用途**：代码段高精度时间测量。

**示例代码**：
```cpp
#include <chrono>
#include <iostream>

// 单次高精度时间测量
auto start = std::chrono::high_resolution_clock::now();
// ... 执行代码 ...
auto end = std::chrono::high_resolution_clock::now();
auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
std::cout << "耗时: " << duration.count() << " 微秒" << std::endl;

// 多次测量取平均
const int iterations = 1000;
auto total_time = std::chrono::nanoseconds(0);
for (int i = 0; i < iterations; ++i) {
    auto start = std::chrono::high_resolution_clock::now();
    // ... 执行代码 ...
    auto end = std::chrono::high_resolution_clock::now();
    total_time += std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
}
auto avg_time = total_time / iterations;
std::cout << "平均耗时: " << std::chrono::duration_cast<std::chrono::microseconds>(avg_time).count() << " 微秒" << std::endl;
```

**输出示例**：
```
耗时: 105 微秒
平均耗时: 102 微秒
```

[src: raw/ingested/2技术/性能优化/内存优化-perf-附录：其他性能分析工具.md]
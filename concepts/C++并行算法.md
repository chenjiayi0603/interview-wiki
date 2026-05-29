# C++ 并行算法

See also: [[C++语言特性]], [[C++多线程与并发]]

## 一、执行策略

C++17 引入了并行算法，通过 `std::execution` 命名空间提供不同的执行策略。

```cpp
#include <execution>
#include <algorithm>
#include <vector>

std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

// 顺序执行
std::sort(std::execution::seq, v.begin(), v.end());

// 并行执行（多线程）
std::sort(std::execution::par, v.begin(), v.end());

// 并行+向量化执行
std::sort(std::execution::par_unseq, v.begin(), v.end());
```

| 策略 | 含义 |
|------|------|
| `std::execution::seq` | 串行，与原算法行为一致 |
| `std::execution::par` | 多线程并行（不同线程处理不同区间） |
| `std::execution::par_unseq` | 多线程 + 允许更激进重排/向量化（SIMD） |

## 二、核心知识点

### 1. 并行算法的选择依据
- **数据量**：数据量足够大时，并行化才有意义（通常 > 10,000 元素）
- **CPU核心数**：并行线程数应接近物理核心数
- **缓存友好性**：顺序访问模式更利于缓存命中

### 2. 扩展性测试
- 线程数增加，性能是否线性提升
- 阿姆达尔定律：加速比受限于串行部分

### 3. 伪共享问题
- 多线程访问同一缓存行的不同变量
- 解决方案：缓存行填充（64字节对齐）

```cpp
// 避免伪共享
struct alignas(64) PaddedCounter {
    std::atomic<int> value;
    char padding[60];  // 填充到64字节
};
```

## 三、面试重点

1. **并行算法的选择依据**：数据量、CPU核心数、缓存友好性
2. **扩展性测试**：线程数增加，性能是否线性提升
3. **伪共享问题**：多线程访问同一缓存行

[src: raw/ingested/2技术/cpp/C++性能优化代码复习指南.md]

## Related Pages
- [[C++语言特性]]
- [[C++多线程与并发]]
- [[Intel_TBB]]
- [[OpenMP]]

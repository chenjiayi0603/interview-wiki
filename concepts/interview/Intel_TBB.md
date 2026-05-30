# Intel TBB

See also: [[C++并行算法]], [[C++多线程与并发]]

## 一、概述

Intel Threading Building Blocks (TBB) 是一个 C++ 模板库，用于简化并行编程。它提供了高级的并行算法、并发容器和任务调度器。

## 二、核心知识点

### 1. 并行算法

```cpp
#include <tbb/parallel_for.h>
#include <tbb/parallel_reduce.h>

// 并行 for 循环
tbb::parallel_for(tbb::blocked_range<int>(0, N), [&](const tbb::blocked_range<int>& r) {
    for (int i = r.begin(); i != r.end(); ++i) {
        // 处理 a[i]
    }
});

// 并行归约
int sum = tbb::parallel_reduce(
    tbb::blocked_range<int>(0, N),
    0,
    [&](const tbb::blocked_range<int>& r, int init) -> int {
        for (int i = r.begin(); i != r.end(); ++i) {
            init += a[i];
        }
        return init;
    },
    std::plus<int>()
);
```

### 2. 并发容器

- `tbb::concurrent_vector`：线程安全的动态数组
- `tbb::concurrent_hash_map`：线程安全的哈希表
- `tbb::concurrent_queue`：线程安全的队列

### 3. 任务组

```cpp
#include <tbb/task_group.h>

tbb::task_group g;
g.run([] { /* 任务1 */ });
g.run([] { /* 任务2 */ });
g.wait();  // 等待所有任务完成
```

### 4. 任务窃取调度

TBB 使用 work-stealing scheduler：
- 每个线程有自己的任务队列
- 空闲线程从其他线程队列尾部窃取任务
- 减少线程间竞争，提高负载均衡

### 5. 内存分配优化

```cpp
#include <tbb/scalable_allocator.h>

// 使用 TBB 的可扩展分配器
std::vector<int, tbb::scalable_allocator<int>> v;
```

## 三、TBB vs OpenMP

| 特性 | TBB | OpenMP |
|------|-----|--------|
| 编程模型 | C++ 模板库 | 编译指令 |
| 灵活性 | 更灵活，可组合 | 较简单 |
| 任务调度 | 任务窃取 | 静态/动态调度 |
| 容器支持 | 并发容器 | 无 |
| 学习曲线 | 较陡 | 较平缓 |

## 四、面试重点

1. **TBB vs OpenMP**：TBB更灵活，OpenMP更简单
2. **任务窃取调度**：work-stealing scheduler
3. **内存分配优化**：tbb::scalable_allocator

[src: raw/ingested/2技术/cpp/C++性能优化代码复习指南.md]

## Related Pages
- [[C++并行算法]]
- [[C++多线程与并发]]
- [[OpenMP]]
- [[SIMD指令优化]]

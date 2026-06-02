# 并行计算库

> OpenMP、Intel TBB、std::execution 并行算法 —— 多核并行编程实战。

---

## 一、OpenMP

### 1.1 简介

OpenMP 是**编译指令式**共享内存并行编程 API，通过 `#pragma` 增量式并行化串行代码。

**特点**：简单易用、跨平台、共享内存模型、增量并行化。

### 1.2 基本指令

```cpp
// 并行区域
#pragma omp parallel
{
    printf("Hello from thread %d\n", omp_get_thread_num());
}

// 并行 for 循环
#pragma omp parallel for
for (int i = 0; i < N; ++i) { a[i] = b[i] + c[i]; }

// 归约
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; ++i) { sum += a[i]; }

// 临界区
#pragma omp critical
{ counter++; }
```

### 1.3 数据共享属性

| 属性 | 说明 |
|------|------|
| `shared` | 所有线程共享（默认大部分变量） |
| `private` | 每线程私有副本 |
| `firstprivate` | 私有副本，并初始化为线程进入前的值 |
| `lastprivate` | 私有副本，最后循环的值写回原变量 |
| `reduction` | 归约操作 |

### 1.4 调度策略

```cpp
#pragma omp parallel for schedule(static, 100)  // 静态块，按 100 一组分配
#pragma omp parallel for schedule(dynamic, 100) // 动态分配
#pragma omp parallel for schedule(guided)       // 指数递减
#pragma omp parallel for schedule(auto)         // 编译器自动选择
```

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `static` | 编译时分配，开销最小 | 负载均衡的任务 |
| `dynamic` | 运行时分配，自动均衡 | 负载不均的任务 |
| `guided` | 从大到小递减 | 大量小任务 |
| `auto` | 编译器自动选择 | 不确定时 |

### 1.5 SIMD 指令（OpenMP 4.x+）

```cpp
#pragma omp simd
for (int i = 0; i < N; ++i) { a[i] = b[i] * c[i]; }
```

---

## 二、Intel TBB

### 2.1 核心特点

- **任务窃取调度**（work-stealing）：自动负载均衡，最大化多核利用率
- **高级并行算法**：parallel_for、parallel_reduce、parallel_sort
- **并发容器**：concurrent_vector、concurrent_hash_map、concurrent_queue
- **高效内存分配**：scalable_allocator

### 2.2 核心 API

```cpp
#include <tbb/tbb.h>

// 并行 for
tbb::parallel_for(0, N, [](int i) {
    a[i] = b[i] * c[i];
});

// 并行归约
int sum = tbb::parallel_reduce(
    tbb::blocked_range<int>(0, N),
    0,
    [](const tbb::blocked_range<int>& r, int init) {
        for (int i = r.begin(); i != r.end(); ++i)
            init += a[i];
        return init;
    },
    std::plus<int>()
);

// 并行排序
tbb::parallel_sort(v.begin(), v.end());

// 任务组
tbb::task_group tg;
tg.run([] { /* task1 */ });
tg.run([] { /* task2 */ });
tg.wait();
```

### 2.3 TBB vs C++ 标准库

| 对比项 | TBB | C++ 标准库 |
|--------|-----|-----------|
| 任务调度 | work-stealing，自动负载均衡 | 仅提供线程/锁原语 |
| 并发容器 | concurrent_vector/map/queue | 无标准并发容器 |
| 内存分配 | scalable_allocator 优化多线程 | new/malloc 竞争严重 |
| 嵌套并行 | 天然支持 | 需手动管理 |
| 依赖 | 需额外链接 | 内置 |

### 2.4 scalable_allocator

```cpp
// 多线程环境下减少锁竞争
std::vector<int, tbb::scalable_allocator<int>> v;
TBB 的内存分配器会为每个线程预分配缓存，减少全局锁竞争。

---

## 三、std::execution 并行算法（C++17）

```cpp
#include <execution>

// 并行执行策略
std::sort(std::execution::par, v.begin(), v.end());        // 并行
std::for_each(std::execution::par_unseq, v.begin(), v.end(), f); // 并行+向量化
std::transform(std::execution::par, in.begin(), in.end(), out.begin(), f);
```

**注意**：实现是否真正并行取决于标准库实现（MSVC 支持好，GCC/libstdc++ 支持有限）。

---

## 四、选型建议

| 场景 | 推荐 |
|------|------|
| 简单并行循环 | OpenMP（最简单） |
| 复杂任务调度 | TBB（最灵活） |
| 标准兼容 | C++17 parallel algorithms |
| 极致性能 | TBB + 手工优化 |

**TBB 优势场景**：需要极致多核性能、复杂任务编排、并发容器、多线程内存分配优化。
**标准库优势场景**：跨平台、零依赖、简单并行需求。

---

## 五、面试高频追问

### Q1: OpenMP 和 TBB 的区别？
- OpenMP：编译指令驱动，增量并行化串行代码容易，但灵活性差
- TBB：库驱动，任务窃取调度自动负载均衡，适合复杂任务编排
- **结论**：简单循环用 OpenMP，复杂任务用 TBB

### Q2: 什么是 false sharing？如何避免？
- 多线程修改同一缓存行的不同变量，导致缓存行在 CPU 间频繁同步
- 解决：缓存行对齐（`alignas(64)`）、填充 padding、线程私有数据

### Q3: work-stealing 如何工作？
- 每个线程维护一个双端队列
- 线程优先处理自己队列尾部的任务（LIFO，更好的缓存局部性）
- 空闲线程从其他线程队列头部"偷取"任务（FIFO，减少竞争）

### Q4: OpenMP 的 reduction 如何实现？
- 每线程私有副本，局部计算，最后归约合并
- 避免多线程同时写共享变量的竞争

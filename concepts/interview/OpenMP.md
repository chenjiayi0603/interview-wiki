# OpenMP

See also: [[C++并行算法]], [[Intel_TBB]], [[C++多线程与并发]]

## 一、概述

OpenMP (Open Multi-Processing) 是一套支持多平台共享内存并行编程的 API，通过编译指令（pragma）实现并行化，无需大幅修改代码结构。

## 二、核心知识点

### 1. 基本语法

```cpp
#include <omp.h>

// 并行 for 循环
#pragma omp parallel for
for (int i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}

// 并行归约
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; ++i) {
    sum += a[i];
}

// 临界区
#pragma omp critical
{
    // 一次只有一个线程执行
    shared_counter++;
}

// 原子操作
#pragma omp atomic
shared_counter++;
```

### 2. 数据共享属性

| 属性 | 含义 |
|------|------|
| `shared` | 所有线程共享同一变量 |
| `private` | 每个线程拥有私有副本 |
| `firstprivate` | 私有副本，并用主线程值初始化 |
| `lastprivate` | 私有副本，最后迭代的值写回 |
| `reduction` | 归约操作（+、*、max、min 等） |

### 3. 调度策略

```cpp
// 静态调度：编译时分配迭代块
#pragma omp parallel for schedule(static, chunk_size)

// 动态调度：运行时动态分配
#pragma omp parallel for schedule(dynamic, chunk_size)

// 引导调度：块大小逐渐减小
#pragma omp parallel for schedule(guided, chunk_size)
```

| 策略 | 特点 | 适用场景 |
|------|------|----------|
| `static` | 固定分配，开销最小 | 负载均衡的任务 |
| `dynamic` | 动态分配，开销较大 | 负载不均衡的任务 |
| `guided` | 折中方案 | 通用场景 |

### 4. 编译与运行

```bash
# 编译（GCC）
g++ -fopenmp program.cpp -o program

# 设置线程数
export OMP_NUM_THREADS=4
./program
```

## 三、面试重点

1. **编译指令方式**：无需修改代码结构
2. **数据竞争**：private/shared/reduction 的使用
3. **调度策略**：static/dynamic/guided 的区别与选择

[src: raw/ingested/2技术/cpp/C++性能优化代码复习指南.md]

## Related Pages
- [[C++并行算法]]
- [[Intel_TBB]]
- [[C++多线程与并发]]
- [[SIMD指令优化]]
- [[性能优化]]
- [[NUMA架构]]

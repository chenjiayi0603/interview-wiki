# OpenMP 调度策略

## schedule 子句

**语法**：
```cpp
#pragma omp parallel for schedule(kind[, chunk_size])
```

## 调度类型

| 调度类型 | 说明 | 适用场景 |
|---------|------|---------|
| **static** | 静态分配，编译时确定 | 迭代工作量均匀 |
| **dynamic** | 动态分配，运行时分配 | 迭代工作量不均匀 |
| **guided** | 引导式，块大小递减 | 迭代工作量不均匀 |
| **auto** | 编译器自动选择 | 让编译器决定 |
| **runtime** | 运行时通过环境变量设置 | 灵活配置 |

## static 调度

```cpp
#pragma omp parallel for schedule(static)
for (int i = 0; i < 100; i++) {
    // 每个线程分配固定数量的迭代
}
```

**chunk_size 示例**：
```cpp
// 完整例子：演示 static 调度，每个线程处理 10 个迭代

#include <stdio.h>
#include <omp.h>

int main() {
    // 每 10 个迭代为一块，按 static 策略静态分配给各线程
    #pragma omp parallel for schedule(static, 10)
    for (int i = 0; i < 100; i++) {
        printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
    }
    return 0;
}
```

**分配方式**（4 线程，100 迭代，chunk=10）：
- Thread 0: 0-9, 40-49, 80-89
- Thread 1: 10-19, 50-59, 90-99
- Thread 2: 20-29, 60-69
- Thread 3: 30-39, 70-79

## dynamic 调度

```cpp
// 完整例子：演示 dynamic 调度，线程动态分配迭代任务

#include <stdio.h>
#include <omp.h>

int main() {
    // dynamic 与 static 最大的区别在于任务的分配方式：
    // static 是线程在一开始就平均（或按 chunk_size）把所有迭代分割好，每个线程分配到固定的一组块，后续不会再变；
    // dynamic 是每当某线程完成一组迭代就动态申请下一组，线程总是“抢”未完成的任务。这样适合每个迭代执行时间差异较大的场景（比如有的循环体很快，有的很慢），可以更好负载均衡，但调度开销略大。
    #pragma omp parallel for schedule(dynamic)
    for (int i = 0; i < 100; i++) {
        printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
    }
    return 0;
}
```

**chunk_size 示例**：
```cpp
// 完整例子：dynamic调度，每次每个线程分配5个迭代

#include <stdio.h>
#include <omp.h>

int main() {
    // 不是只能接for，可以接sections、single、parallel等OpenMP支持的block型结构
    // 但 schedule(dynamic, 5) 只用于for循环，作用是指定循环调度方式
    #pragma omp parallel for schedule(dynamic, 5)
    for (int i = 0; i < 100; i++) {
        printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
    }
    return 0;
}
```

**特点**：
- 适合迭代工作量不均匀的情况
- 负载均衡好，但调度开销较大

## guided 调度

```cpp
// 完整例子：演示 guided 调度的效果，任务块大小自动递减

#include <stdio.h>
#include <omp.h>

int main() {
    // guided: 初始分配大块，未完成部分自动减小块大小分配，实现负载均衡且减少调度开销
    #pragma omp parallel for schedule(guided)
    for (int i = 0; i < 100; i++) {
        printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
    }
    return 0;
}
```

**特点**：
- 初始块较大，后续块逐渐减小
- 平衡了负载均衡和调度开销

## 调度策略选择

**选择指南**：

```cpp
// 1. 工作量均匀 → static
#pragma omp parallel for schedule(static)
for (int i = 0; i < n; i++) {
    uniform_work(i);
}

// 2. 工作量不均匀 → dynamic
#pragma omp parallel for schedule(dynamic, 1)
for (int i = 0; i < n; i++) {
    variable_work(i);
}

// 3. 不确定 → guided 或 auto
#pragma omp parallel for schedule(guided)
for (int i = 0; i < n; i++) {
    unknown_work(i);
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-调度策略.md]
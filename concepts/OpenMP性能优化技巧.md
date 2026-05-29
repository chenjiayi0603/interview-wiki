# OpenMP 性能优化技巧

> 本文总结 OpenMP 并行编程中的性能优化技巧，包括减少线程创建开销、避免 false sharing、最小化临界区、合理使用 nowait、数据局部性优化、线程绑定和负载均衡。

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 1. 减少线程创建开销

**问题**：频繁创建/销毁线程开销大

**解决方案**：
```cpp
// ❌ 不好：每次循环都创建线程
for (int iter = 0; iter < 1000; iter++) {
    #pragma omp parallel for
    for (int i = 0; i < n; i++) {
        // ...
    }
}

// ✅ 好：在外部创建并行区域
// 每次进入新的并行区（#pragma omp parallel）时，OpenMP 都会重新创建一组线程；离开并行区后，这些线程会被销毁。所以如果多次进入/退出并行区，就会多次创建和销毁线程，带来额外开销。
#pragma omp parallel
{
    for (int iter = 0; iter < 1000; iter++) {
        #pragma omp for
        for (int i = 0; i < n; i++) {
            // ...
        }
    }
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 2. 避免 false sharing

**问题**：不同线程访问同一缓存行的不同变量

**解决方案**：
```cpp
// ❌ 不好：可能 false sharing
int sum[4];  // 假设在同一个缓存行
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    // 问题：可能发生“虚假共享”（false sharing）。
    // sum[omp_get_thread_num()] 对应的 sum[] 元素很可能在同一个缓存行，不同线程同时写入，导致缓存一致性失效、性能降低。
    sum[omp_get_thread_num()] += array[i];
}

// ✅ 好：使用 padding 或 reduction
#pragma omp parallel for reduction(+:total_sum)
for (int i = 0; i < n; i++) {
    total_sum += array[i];
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 3. 最小化临界区

```cpp
// ❌ 不好：整个循环在临界区
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    #pragma omp critical
    {
        result += compute(i);
    }
}

// ✅ 好：先局部计算，再归约
#pragma omp parallel for reduction(+:result)
for (int i = 0; i < n; i++) {
    result += compute(i);
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 4. 合理使用 nowait

```cpp
#pragma omp parallel
{
    // 说明：nowait 表示该 for 循环结束后各线程不需等待，直接进入后面的代码（不进行同步等待）
    #pragma omp for nowait
    for (int i = 0; i < n; i++) {
        process_A(i);
    }
    
    // 不需要等待上面的循环完成
    #pragma omp for
    for (int i = 0; i < m; i++) {
        process_B(i);
    }
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 5. 数据局部性优化

```cpp
// ✅ 好：按行访问（缓存友好）
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) {
        C[i][j] = A[i][j] + B[i][j];
    }
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 6. 线程绑定

```bash
# 绑定线程到 CPU 核心
export OMP_PROC_BIND=true
export OMP_PLACES=cores
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## 7. 负载均衡

```cpp
// 完整例子：根据工作量选择调度策略

#include <stdio.h>
#include <omp.h>

void do_work(int i) {
    // 假定计算量不同
    for (volatile int j = 0; j < (i % 10 + 1) * 10000; ++j);
}

int main() {
    int n = 100;
    int workload_is_uniform = 0; // 修改为1试试均匀调度效果

    /*
    分析为什么这里要根据 workload_is_uniform 选择不同的调度策略：

    - 如果 workload_is_uniform == 1，说明每次 do_work(i) 的计算量都基本一样，此时采用 static 调度（即 schedule(static)），OpenMP 会把所有迭代事先按照数量均分到各个线程上（比如4线程、每人25个任务），这样负载均衡且效率高，开销小。

    - 如果 workload_is_uniform == 0，则不同的 i，do_work(i) 的计算量有差异，某些任务耗时长，某些短，这种情况下，如果还用 static，慢线程会拖累整体执行（因为静态分配了“大任务”）。此时用 dynamic（如 schedule(dynamic, 1)），OpenMP 让线程每次只领取一个任务，谁先做完谁先领取下一个，能让快线程多做工作，实现负载均衡，提高并行效率。

    总结：静态调度适合每个任务工作量均匀，动态调度适合任务不均匀且差异较大。

    */
    #pragma omp parallel
    {
        if (workload_is_uniform) {
            // 静态调度：适合每个循环任务所需工作量差不多的场景
            #pragma omp for schedule(static)
            for (int i = 0; i < n; i++) {
                do_work(i);
                #pragma omp critical
                printf("Thread %d working on i=%d (static)\n", omp_get_thread_num(), i);
            }
        } else {
            // 动态调度：适合任务量不均匀的场景，能自动实现负载均衡
            // 使用动态调度实现负载均衡，让线程根据做完的速度自动领取下一个任务
            #pragma omp for schedule(dynamic, 1)
            for (int i = 0; i < n; i++) {
                do_work(i);
                #pragma omp critical
                printf("Thread %d working on i=%d (dynamic)\n", omp_get_thread_num(), i);
            }
        }
    }

    printf("All work done.\n");
    return 0;
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-性能优化技巧.md]

## Related Pages
- [[OpenMP工作共享结构]]
- [[OpenMP常见问题与调试]]
- [[C++多线程与并发]]
- [[C++并发性能优化]]
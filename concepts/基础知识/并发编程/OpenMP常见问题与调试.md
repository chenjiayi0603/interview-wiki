# OpenMP 常见问题与调试

> OpenMP 并行编程的常见问题、调试方法和性能优化技巧。

See also: [[C++多线程与并发]], [[OpenMP工作共享结构]], [[OpenMP同步机制]], [[OpenMP实战案例分析]]

## 1. 数据竞争（Data Race）

**问题**：多个线程同时修改共享变量

**示例**：
```cpp
// ❌ 有数据竞争
int counter = 0;
#pragma omp parallel for
for (int i = 0; i < 1000; i++) {
    counter++;  // 多个线程同时修改
}
```

**解决方案**：
```cpp
// ✅ 使用 atomic
int counter = 0;
#pragma omp parallel for
for (int i = 0; i < 1000; i++) {
    #pragma omp atomic
    counter++;
}

// ✅ 使用 reduction
int counter = 0;
#pragma omp parallel for reduction(+:counter)
for (int i = 0; i < 1000; i++) {
    counter++;
}
```

## 2. 死锁

**问题**：多个临界区嵌套导致死锁

**示例**：
```cpp
// ❌ 可能死锁
#pragma omp parallel sections
{
    #pragma omp section
    {
        #pragma omp critical(A)
        {
            #pragma omp critical(B)
            {
                // ...
            }
        }
    }
    
    #pragma omp section
    {
        #pragma omp critical(B)
        {
            #pragma omp critical(A)  // 死锁！
            {
                // ...
            }
        }
    }
}
```

**解决方案**：统一锁的顺序

## 3. 性能不提升

**可能原因**：
1. 并行开销大于计算量
2. 负载不均衡
3. False sharing
4. 过多的同步操作

### 3.1 并行开销大于计算量

**例子**：
```cpp
// 只是累加10个元素，并行线程启动和同步的成本大于实际计算
int sum = 0;
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < 10; i++) {
    sum += i;
}
```
这类任务量太小，反而比串行更慢。

### 3.2 负载不均衡

**例子**：
```cpp
// 前半部分循环每次很快，后半部分很慢，线程间工作量不一致
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    if (i < n/2)
        quick_task(i);
    else
        slow_task(i);
}
```
导致有的线程很早完成，部分线程拖慢整体进度。

### 3.3 False sharing

**例子**：
```cpp
int arr[16] = {0};
#pragma omp parallel for
for (int i = 0; i < 16; i++) {
    arr[i]++;  // 多线程分别操作相邻元素，导致缓存行伪共享
}
```
多线程更新相邻数组元素时，频繁触发缓存同步，严重影响性能。

### 3.4 过多的同步操作

**例子**：
```cpp
int sum = 0;
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    #pragma omp critical
    {
        sum += i;  // 每次累加都需要锁，极大拖慢速度
    }
}
```
频繁进入临界区或 barrier，会显著降低并行效率。

**调试方法**：
```cpp
// 测量并行开销
double start = omp_get_wtime();
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    // 计算
}
double end = omp_get_wtime();
printf("Parallel time: %f\n", end - start);
```

## 4. 线程数设置

```cpp
// 动态调整线程数
int optimal_threads = omp_get_num_procs();
omp_set_num_threads(optimal_threads);

#pragma omp parallel
{
    // ...
}
```

## 5. 调试工具

**使用线程检查器**：
- Intel Inspector
- ThreadSanitizer (TSan)
- Helgrind (Valgrind)

**编译选项**：
```bash
# ThreadSanitizer
gcc -fopenmp -fsanitize=thread program.c

# 运行时检查
export TSAN_OPTIONS="detect_deadlocks=1"
```

## 6. 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| 结果不正确 | 数据竞争 | 使用 reduction 或 critical |
| 性能下降 | 并行开销大 | 增加计算量或减少线程数 |
| 段错误 | 数组越界 | 检查循环边界 |
| 死锁 | 锁顺序问题 | 统一锁的获取顺序 |

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-常见问题与调试.md]
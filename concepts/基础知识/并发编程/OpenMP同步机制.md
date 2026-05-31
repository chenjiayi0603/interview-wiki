# OpenMP 同步机制

> OpenMP 提供了多种同步机制用于并行区域的线程协调。

See also: [[C++多线程与并发]], [[线程同步机制]], [[POSIX线程管理]]

## barrier

**显式屏障**：
```cpp
#pragma omp barrier
```

**隐式屏障**：
- `parallel`、`for`、`sections`、`single` 结束时都有隐式 barrier
- 使用 `nowait` 子句可以取消隐式 barrier

**示例**：
```cpp
// 完整例子：展示如何用 barrier 显式同步线程在两个阶段

#include <omp.h>
#include <stdio.h>

void phase1(int tid) {
    printf("Thread %d: phase1 doing work\n", tid);
    // ...模拟工作
}

void phase2(int tid) {
    printf("Thread %d: phase2 doing work\n", tid);
    // ...模拟工作
}

int main() {
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();

        phase1(tid);

        #pragma omp barrier  // 显式同步：确保所有线程都完成 phase1

        phase2(tid);
    }
    return 0;
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-同步机制.md]

## critical

**语法**：
```cpp
// 完整例子：不同线程累加结果到共享变量，使用无名和命名 critical

#include <omp.h>
#include <stdio.h>

void critical_section() {
    // 模拟某些只允许单线程访问的操作
}

int main() {
    int sum = 0;

    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        int value = tid + 1; // 每个线程计算一个值

        // 无名临界区：确保 critical_section() 只被一个线程执行
        #pragma omp critical
        {
            printf("Thread %d in unnamed critical section\n", tid);
            critical_section();
        }

        // 命名临界区：线程安全累加到 sum
        #pragma omp critical(update_sum) // update_sum 是临界区的名字，不同名字的 critical 可以并发执行，相同名字则互斥
        {
            sum += value;
            printf("Thread %d added %d to sum, current sum = %d\n", tid, value, sum);
        }
    }

    printf("Final sum = %d\n", sum);
    return 0;
}
```

**互斥锁（mutex）与 OpenMP 的 critical 指令本质类似**，都是保证同一时刻临界区只被一个线程执行，用于同步共享数据。
但二者区别如下：

1. **写法与层次**
   - mutex 是底层 C/C++ 的锁（如 std::mutex/pthread_mutex_t），需要手动 lock()/unlock()。
   - critical 是 OpenMP 的高层同步指令，不需要手动声明锁，由编译器自动生成加锁/解锁代码。

2. **粒度和作用范围**
   - mutex 可以锁任意作用域，也可用于 OpenMP 外其它线程同步。
   - critical 只能用于 OpenMP 并行区内的同步，粒度以代码块为单位。

3. **灵活性**
   - mutex 可跨不同并发库使用（如和 std::thread/pthread 结合），适用范围广。
   - critical 只能控制 OpenMP 线程间的同步，和外部线程体系不可混用。

4. **性能**
   - 二者性能类似（都依赖底层锁实现），但 critical 适合简单同步，复杂场景下 mutex 更灵活高效。

5. **命名临界区**
   - OpenMP 支持 critical(name) 命名临界区，不同名字间可并发，mutex 靠多把锁实现类似效果。

6. **调试与风险**
   - 使用 mutex 不当容易死锁，critical 死锁风险较低（自动加锁解锁）。

总结：critical 用起来最方便、跨平台，首选 OpenMP 并行同步。需要高级用法或跨 OpenMP 范围请用 mutex。

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-同步机制.md]

**示例：线程安全的计数器**
```cpp
int counter = 0;

#pragma omp parallel
{
    for (int i = 0; i < 1000; i++) {
        #pragma omp critical
        {
            counter++;  // 原子操作
        }
    }
}
```

**命名临界区**：
```cpp
#pragma omp critical(update_counter)
{
    counter++;
}

#pragma omp critical(update_sum)
{
    sum += value;
}
// 两个临界区可以并行执行
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-同步机制.md]

## atomic

**语法**：
```cpp
#pragma omp atomic
statement;
```

**支持的原子操作**：
- `x++`, `x--`, `++x`, `--x`
- `x binop= expr`（binop: +, -, *, /, &, ^, |, <<, >>）
- `x = x binop expr`
- `x = expr binop x`

**示例**：
```cpp
int counter = 0;

#pragma omp parallel for
for (int i = 0; i < 1000; i++) {
    #pragma omp atomic
    counter++;
}
```

**atomic vs critical**：
- `atomic`：只适用于简单的原子操作，性能更好
- `critical`：适用于任意代码块，更灵活但性能较差

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-同步机制.md]

## flush

**语法**：
```cpp
#pragma omp flush [variable-list]
```

**作用**：确保内存一致性，使所有线程看到变量的最新值

**示例**：
```cpp
int flag = 0;
int data = 0;

#pragma omp parallel sections
{
    #pragma omp section
    {
        data = 42;
        #pragma omp flush(data)
        flag = 1;
        #pragma omp flush(flag)
    }
    
    #pragma omp section
    {
        while (flag == 0) {
            #pragma omp flush(flag)
        }
        #pragma omp flush(data)
        printf("data = %d\n", data);  // 保证看到 42
    }
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-同步机制.md]

## ordered

**语法**：
```cpp
#pragma omp parallel for ordered
for (int i = 0; i < n; i++) {
    // 无序部分
    process(i);
    
    #pragma omp ordered
    {
        // 按迭代顺序执行
        printf("Iteration %d\n", i);
    }
}
```

**用途**：在并行循环中保持部分代码的顺序执行

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-同步机制.md]
# OpenMP 工作共享结构

> 本文介绍 OpenMP 中的工作共享结构（work-sharing constructs），包括 `parallel for`、`sections`、`single`、`master` 等。

See also: [[C++多线程与并发]], [[现代C++特性按版本划分]]

---

## parallel for

### 基本用法
```cpp
#pragma omp parallel for
for (int i = 0; i < n; i++) {
    // 循环体
}
```

### 等价写法
```cpp
#pragma omp parallel
{
    #pragma omp for
    for (int i = 0; i < n; i++) {
        // 循环体
    }
}
```

### 限制条件
- 循环变量必须是整数或指针类型
- 循环的起始值、结束值、步长必须是循环不变量
- 不能有 `break`、`goto`、`return`（除非在循环体内）
- 循环必须是规范形式：`for (init; test; incr)`

### 示例：向量加法
```cpp
void vector_add(double *a, double *b, double *c, int n) {
    #pragma omp parallel for
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}
```

## sections

sections 与 omp parallel for 的区别如下：
1. 用途不同：sections 适用于并行执行多个不同的任务（每个 section 可以是完全不同的代码块），而 parallel for 适用于并行拆分同一个 for 循环的不同迭代（任务基本一致，仅数据不同）。
2. 并发工作方式：parallel for 会将循环的迭代分给多个线程，每个线程处理部分迭代；sections 则是每个 section 分配给不同线程，同时运行各自的代码块。
3. 实际场景：当有多个独立的计算、IO 或处理任务要并行时用 sections；当有一个大循环任务（如矩阵、数组运算）时用 parallel for。
4. 语法结构：sections 语法需要使用 #pragma omp section 明确标记每段；parallel for 直接针对 for 循环体。

### 语法
```cpp
#pragma omp parallel sections
{
    #pragma omp section
    {
        // 代码段 1
    }
    
    #pragma omp section
    {
        // 代码段 2
    }
    
    #pragma omp section
    {
        // 代码段 3
    }
}
```

### 示例：并行执行多个任务
```cpp
// 完整例子：并行执行 3 个独立计算任务

#include <stdio.h>
#include <omp.h>

void compute_A() {
    printf("Section 1: Computing A, thread %d\n", omp_get_thread_num());
    // 假设是一些计算
    for (int i = 0; i < 100000000; ++i); // 占位计算
}

void compute_B() {
    printf("Section 2: Computing B, thread %d\n", omp_get_thread_num());
    for (int i = 0; i < 100000000; ++i);
}

void compute_C() {
    printf("Section 3: Computing C, thread %d\n", omp_get_thread_num());
    for (int i = 0; i < 100000000; ++i);
}

int main() {
    // 开启多线程并行 sections，各 section 执行不同任务
    #pragma omp parallel sections
    {
        #pragma omp section
        // 一个 section 通常由一个线程执行，多个 section 会被不同线程并行分配。具体每个 section 是否一定由单独线程执行，取决于可用线程数与 section 数量；
        // 如果 section 数量 > 线程数，则有些线程会连续执行多个 section。
        // 多数情况下，每个 section 会被1个线程独立并发执行。
        {
            compute_A();
        }
        
        #pragma omp section
        {
            compute_B();
        }
        
        #pragma omp section
        {
            compute_C();
        }
    }
    printf("All sections finished.\n");
    return 0;
}
```

## single

### 语法
```cpp
#pragma omp parallel
{
    // 所有线程执行
    do_work();
    
    #pragma omp single
    {
        // 只有一个线程执行
        printf("This is executed by one thread\n");
    }
    
    // 所有线程继续执行（有隐式 barrier）
}
```

### 示例：初始化操作
```cpp
#pragma omp parallel
{
    #pragma omp single
    {
        printf("Initializing...\n");
        initialize_data();
    }
    
    // 所有线程等待初始化完成后再继续
    process_data();
}
```

## master

### 语法
```cpp
// 举例：只让主线程执行某段逻辑，其它线程不执行
#pragma omp parallel
{
    printf("Thread %d: before master section\n", omp_get_thread_num());

    #pragma omp master
    {
        // 只有主线程（thread 0）会执行此处
        printf("Thread %d: only master executes this\n", omp_get_thread_num());
    }

    printf("Thread %d: after master section\n", omp_get_thread_num());
}
```

### 与 single 的区别
- `single`：任意一个线程执行，有隐式 barrier
- `master`：只有主线程执行，无隐式 barrier

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-工作共享结构.md]
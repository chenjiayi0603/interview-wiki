# OpenMP 实战案例分析

> 本文包含 OpenMP 并行编程的多个实战案例，涵盖蒙特卡洛计算 π、矩阵乘法、归并排序、图像处理、数值积分等场景。

See also: [[C++多线程与并发]], [[OpenMP-vs-线程池]]

## 案例 1：并行计算 π（蒙特卡洛方法）

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

// 串行计算圆周率π的近似值：蒙特卡洛采样法
double compute_pi_serial(int n_samples) {
    int count = 0;
    double x, y;
    unsigned int seed = 12345;
    for (int i = 0; i < n_samples; i++) {
        x = (double)rand_r(&seed) / RAND_MAX;
        y = (double)rand_r(&seed) / RAND_MAX;
        if (x * x + y * y <= 1.0) {
            count++;
        }
    }
    return 4.0 * count / n_samples;
}

// 使用OpenMP并行计算圆周率π的近似值
// 利用OpenMP多线程并行计算圆周率π的近似值（蒙特卡洛法）
double compute_pi_parallel(int n_samples) {
    int count = 0;      // 记录落在圆内的点数
    double x, y;        // 随机点的坐标
    int actual_threads = 0; // 实际用到的线程数

    // 并行区域，每个线程独立生成随机点统计落圆内的数量
    
    #pragma omp parallel private(x, y) reduction(+:count)
    {
        // 统计实际用到的线程数
        #pragma omp atomic
        actual_threads++;
        print_thread_info();
        
        // 为每个线程初始化一个不同的随机数种子，避免竞争
        unsigned int seed = omp_get_thread_num() + 1234;

        // 将循环任务自动拆分给各线程处理
        #pragma omp for
        for (int i = 0; i < n_samples; i++) {
            // 生成[0,1]范围内的随机点
            x = (double)rand_r(&seed) / RAND_MAX;
            y = (double)rand_r(&seed) / RAND_MAX;
            // 判断是否在单位圆内
            if (x * x + y * y <= 1.0) {
                count++;    // 落在圆内则计数
            }
        }
    }
    printf("compute_pi_parallel 实际用到的线程数: %d\n", actual_threads);
    // 计算π的近似值：π ≈ 4 × 圆内点数 / 总点数
    return 4.0 * count / n_samples;
}

/*
Q: 要是compute_pi_parallel 调用次数过高，是否会频繁创建销毁线程？

A: 如果 compute_pi_parallel 被频繁调用，那么每次调用时通常都会创建和销毁 OpenMP 的线程池。但是，大多数主流 OpenMP 实现（如GCC/libgomp、Intel OpenMP等）为了减小线程创建开销，会采取“线程池”复用机制——即第一次进入并行区时会创建所需线程，之后的并行区大多数情况下会复用这些线程，而不是每次都重新创建和销毁。这样可以显著减少线程操作带来的性能损耗。线程的真正销毁通常只会在程序结束或执行 omp_set_num_threads 等重大配置变更后才发生。

只有在OpenMP实现没有线程池复用，或者手动调用如 omp_set_dynamic(1)、频繁更改线程数等特殊场景下，才有可能比较频繁地创建和销毁线程。但这种情况在常规用法下并不常见。

总结：正常情况下 compute_pi_parallel 多次调用不会频繁创建销毁线程，OpenMP会自动优化，大多数时候都采用线程复用以提升性能。
*/

void test1() {
    // 验证 compute_pi_parallel 是否使用线程池机制：
    // 思路：多次调用 compute_pi_parallel，并输出每次的 "实际用到的线程数"，
    // 如果每次用到的线程数保持一致（且不频繁增加/减少），且程序没有变慢，通常说明OpenMP实现有用到线程池机制。

    printf("\n验证 compute_pi_parallel 是否复用线程池:\n");
    int repeat = 5;
    for (int i = 0; i < repeat; i++) {
        double t0 = omp_get_wtime();
        double pi_inner = compute_pi_parallel(n);
        double t1 = omp_get_wtime();
        printf("第%d次: π ≈ %.8f, 用时: %.6f 秒\n", i + 1, pi_inner, t1 - t0);
    }
}

int main() {
    int n = 10000000;
    double pi;
    double t_start, t_end;

    // 串行时间
    t_start = omp_get_wtime();
    pi = compute_pi_serial(n);
    t_end = omp_get_wtime();
    printf("串行计算: π ≈ %.8f\n", pi);
    printf("串行用时: %.6f 秒\n", t_end - t_start);

    // 并行时间
    // 打印OpenMP用到的线程数
    #pragma omp parallel
    {
        #pragma omp single
        {
            printf("OpenMP使用的线程数: %d\n", omp_get_num_threads());
        }
    }
    t_start = omp_get_wtime();
    pi = compute_pi_parallel(n);
    t_end = omp_get_wtime();
    printf("OpenMP并行计算: π ≈ %.8f\n", pi);
    printf("并行用时: %.6f 秒\n", t_end - t_start);

    test1();
    return 0;
}

// cd /home/chen/cpp-test1 && g++-11 -std=c++20 -fopenmp test1.cpp -o test1 && ./test1

```

串行计算: π ≈ 3.14138640
串行用时: 0.078873 秒
OpenMP使用的线程数: 28
compute_pi_parallel 实际用到的线程数: 28
OpenMP并行计算: π ≈ 3.14121960
并行用时: 0.016010 秒

### 线程池复用机制

OpenMP 实现（如 GCC/libgomp、Intel OpenMP）通常采用线程池复用机制：第一次进入并行区时创建所需线程，之后的并行区复用这些线程，而不是每次都重新创建和销毁。线程的真正销毁通常只会在程序结束或执行 `omp_set_num_threads` 等重大配置变更后才发生。

## 案例 2：并行矩阵乘法

```cpp
void parallel_matrix_multiply(double **A, double **B, double **C, int n) {
    #pragma omp parallel
    {
        int i, j, k;
        double sum;
        
        #pragma omp for schedule(static)
        for (i = 0; i < n; i++) {
            for (j = 0; j < n; j++) {
                sum = 0.0;
                for (k = 0; k < n; k++) {
                    sum += A[i][k] * B[k][j];
                }
                C[i][j] = sum;
            }
        }
    }
}
```

## 案例 3：并行归并排序

```cpp
void merge(int *arr, int left, int mid, int right) {
    // 归并操作
}

void merge_sort_parallel(int *arr, int left, int right) {
    if (left < right) {
        int mid = (left + right) / 2;
        
        #pragma omp parallel sections
        {
            #pragma omp section
            {
                merge_sort_parallel(arr, left, mid);
            }
            
            #pragma omp section
            {
                merge_sort_parallel(arr, mid + 1, right);
            }
        }
        
        merge(arr, left, mid, right);
    }
}
```

## 案例 4：并行图像处理

```cpp
void parallel_image_filter(unsigned char *image, int width, int height) {
    unsigned char *output = malloc(width * height);
    
    #pragma omp parallel for
    for (int y = 1; y < height - 1; y++) {
        for (int x = 1; x < width - 1; x++) {
            // 3x3 卷积核
            int sum = 0;
            for (int dy = -1; dy <= 1; dy++) {
                for (int dx = -1; dx <= 1; dx++) {
                    sum += image[(y + dy) * width + (x + dx)];
                }
            }
            output[y * width + x] = sum / 9;
        }
    }
    
    free(output);
}
```

## 案例 5：并行数值积分

```cpp
double integrate(double (*f)(double), double a, double b, int n) {
    double h = (b - a) / n;
    double sum = 0.0;
    
    #pragma omp parallel for reduction(+:sum)
    for (int i = 0; i < n; i++) {
        double x = a + (i + 0.5) * h;
        sum += f(x);
    }
    
    return h * sum;
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-实战案例分析.md]
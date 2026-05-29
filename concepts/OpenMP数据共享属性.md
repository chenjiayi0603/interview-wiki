# OpenMP 数据共享属性

## 数据共享类别

| 属性 | 说明 | 默认规则 |
|------|------|---------|
| **shared** | 所有线程共享同一变量 | 并行区域外的变量默认 shared |
| **private** | 每个线程有独立副本 | 循环变量、临时变量默认 private |
| **firstprivate** | private + 用原值初始化 | - |
| **lastprivate** | private + 最后迭代值赋回原变量 | - |
| **reduction** | 归约操作（求和、求积等） | - |

## private 子句

```cpp
int x = 10;
#pragma omp parallel private(x)
{
    x = omp_get_thread_num();  // 每个线程有独立的 x
    printf("Thread %d: x = %d\n", omp_get_thread_num(), x);
}
// 并行区域外，x 仍为 10（未修改）
```

### 示例：矩阵乘法

```cpp
#include <vector>
#include <stdio.h>
#include <omp.h>
using std::vector;

void matrix_multiply(vector<vector<double>>& A, vector<vector<double>>& B, vector<vector<double>>& C, int n) {
    int i, j, k;
    double sum;

    #pragma omp parallel
    {
        #pragma omp single
        {
            int t = omp_get_num_threads();
            printf("实际使用线程数: %d\n", t);
        }
    }

    #pragma omp parallel for private(i, j, k, sum) shared(A, B, C)
    for (i = 0; i < n; i++) {
        int thread_id = omp_get_thread_num();
        printf("线程 %d 负责计算第 %d 行\n", thread_id, i);
        for (j = 0; j < n; j++) {
            sum = 0.0;
            for (k = 0; k < n; k++) {
                sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
        }
    }
}

int main() {
    int n = 4, i, j;
    vector<vector<double>> A(n, vector<double>(n));
    vector<vector<double>> B(n, vector<double>(n));
    vector<vector<double>> C(n, vector<double>(n));

    for (i = 0; i < n; i++)
        for (j = 0; j < n; j++) {
            A[i][j] = i + j;
            B[i][j] = (i == j) ? 1.0 : 0.0;
        }

    matrix_multiply(A, B, C, n);

    printf("矩阵C = A * B:\n");
    for (i = 0; i < n; i++) {
        for (j = 0; j < n; j++) {
            printf("%6.2f ", C[i][j]);
        }
        printf("\n");
    }

    return 0;
}
```

## firstprivate 子句

```cpp
int x = 5;
#pragma omp parallel firstprivate(x)
{
    x += omp_get_thread_num();  // 每个线程从 5 开始
    printf("Thread %d: x = %d\n", omp_get_thread_num(), x);
}
```

## lastprivate 子句

```cpp
int last_i;
#pragma omp parallel for lastprivate(last_i)
for (int i = 0; i < 100; i++) {
    last_i = i;
    // 处理...
}
// last_i 将是最后一个迭代的值（99）
```

## reduction 子句

**支持的归约操作符**：
- `+`, `-`, `*`, `&`, `|`, `^`
- `&&`, `||`
- `min`, `max`

**语法**：
```cpp
#pragma omp parallel for reduction(op:variable)
```

### 示例：数组求和

```cpp
int sum = 0;
int array[1000];

for (int i = 0; i < 1000; i++) {
    array[i] = i;
}

#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < 1000; i++) {
    sum += array[i];
}

printf("Sum = %d\n", sum);  // 输出: Sum = 499500
```

### 示例：求最大值

```cpp
int max_val = INT_MIN;
int array[1000];

#pragma omp parallel for reduction(max:max_val)
for (int i = 0; i < 1000; i++) {
    if (array[i] > max_val) {
        max_val = array[i];
    }
}
```

### 示例：点积计算

```cpp
double dot_product = 0.0;
double a[1000], b[1000];

#pragma omp parallel for reduction(+:dot_product)
for (int i = 0; i < 1000; i++) {
    dot_product += a[i] * b[i];
}
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-数据共享属性.md]
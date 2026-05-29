# SIMD 指令优化

See also: [[C++并行算法]], [[性能优化]], [[C++内联汇编]]

## 一、概述

SIMD (Single Instruction, Multiple Data) 是一种数据并行技术，一条指令同时操作多个数据。Intel 提供了多种 SIMD 指令集扩展。

## 二、核心知识点

### 1. SIMD 指令集演进

| 指令集 | 寄存器宽度 | 引入时间 | 说明 |
|--------|-----------|----------|------|
| MMX | 64 位 | 1997 | 整数运算 |
| SSE | 128 位 | 1999 | 单精度浮点 |
| SSE2/3/4 | 128 位 | 2001-2006 | 双精度、整数扩展 |
| AVX | 256 位 | 2011 | `__m256` 类型 |
| AVX2 | 256 位 | 2013 | 整数运算扩展 |
| AVX-512 | 512 位 | 2016 | 服务器/高性能计算 |

### 2. Intel Intrinsics 基础

```cpp
#include <immintrin.h>

// 256 位向量类型
__m256  // 8 个 float
__m256d // 4 个 double
__m256i // 整数向量

// 基本操作示例：向量加法
void vector_add(const float* a, const float* b, float* c, int n) {
    for (int i = 0; i < n; i += 8) {
        __m256 va = _mm256_load_ps(&a[i]);   // 加载 8 个 float
        __m256 vb = _mm256_load_ps(&b[i]);
        __m256 vc = _mm256_add_ps(va, vb);   // 一次加 8 个
        _mm256_store_ps(&c[i], vc);          // 存储结果
    }
}
```

### 3. 内存对齐

```cpp
// AVX 需要 32 字节对齐
float* data = (float*)aligned_alloc(32, N * sizeof(float));

// 或使用 alignas
alignas(32) float data[N];

// 对齐加载（更快）
__m256 v = _mm256_load_ps(data);

// 非对齐加载（兼容但稍慢）
__m256 v = _mm256_loadu_ps(data);
```

### 4. 自动向量化

编译器（GCC/Clang）在优化级别 `-O2` 或 `-O3` 下会自动尝试向量化简单循环：

```cpp
// 编译器可能自动向量化的代码特征：
// - 简单循环，无复杂控制流
// - 连续内存访问
// - 无循环依赖
for (int i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}

// 编译时查看向量化报告
g++ -O3 -fopt-info-vec program.cpp
```

## 三、适用场景

- **数组运算**：向量加减、点积
- **图像处理**：像素操作、滤镜
- **矩阵运算**：矩阵乘法
- **信号处理**：FFT、卷积
- **科学计算**：数值模拟

## 四、面试重点

1. **SIMD适用场景**：数组运算、图像处理、矩阵运算
2. **内存对齐**：32字节对齐 (AVX)
3. **自动向量化**：编译器能自动优化哪些代码

[src: raw/ingested/2技术/cpp/C++性能优化代码复习指南.md]

## Related Pages
- [[C++并行算法]]
- [[性能优化]]
- [[C++内联汇编]]
- [[OpenMP]]
- [[Intel_TBB]]
- [[C++语言特性]]

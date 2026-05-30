# CPU优化

## 4.1 分支预测优化

### 4.1.1 减少分支

```cpp
// ❌ 多个分支
int get_value(int type) {
    if (type == 1) return 10;
    if (type == 2) return 20;
    if (type == 3) return 30;
    return 0;
}

// ✅ 使用查找表
int get_value(int type) {
    static const int values[] = {0, 10, 20, 30};
    if (type >= 0 && type < 4) {
        return values[type];
    }
    return 0;
}

// ✅ 使用位运算代替分支
// 原始代码
int abs_value(int x) {
    if (x < 0) return -x;
    return x;
}

// 优化后（无分支）
int abs_value(int x) {
    int mask = x >> 31;  // 如果 x < 0，mask = -1，否则 mask = 0
    return (x ^ mask) - mask;
}
```

### 4.1.2 分支预测提示

```cpp
// 使用 likely/unlikely 提示编译器
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

int process(int value) {
     // 这里性能好的原因：CPU分支预测器能提升常见分支效率。即使预测失误顶多导致流水线清空和性能下降，不会出现逻辑错误。
    if (likely(value > 0)) { 
        return value * 2;
    } else {
        return 0;
    }
}

// C++20 支持
if (value > 0) [[likely]] {
    return value * 2;
} else [[unlikely]] {
    return 0;
}
```

### 4.1.3 条件移动指令

```cpp
// 编译器可能将简单分支优化为条件移动（cmov）
int max_value(int a, int b) {
    return a > b ? a : b;  // 可能优化为 cmov 指令，无分支
}
```

## 4.2 SIMD向量化

### 4.2.1 自动向量化

```cpp
// 编译器可能自动向量化
void add_arrays(int* a, int* b, int* c, int n) {
    for (int i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];  // 可能自动使用 SIMD
    }
}

// 帮助编译器向量化
void add_arrays(int* __restrict__ a, int* __restrict__ b, int* __restrict__ c, int n) {
    // __restrict__ 告诉编译器指针不重叠
    // OpenMP 库的指令，告诉编译器对这段循环进行 SIMD 向量化
    // simd 是 “单指令多数据”（Single Instruction Multiple Data）的缩写，指一次操作可以并行处理多组数据（向量化运算）
    // 没有调用线程池，这里用的是 OpenMP 的 simd 指令（自动向量化），并非线程池并行。
     // 为啥快：让编译器将循环自动向量化，将一组数据并行处理，多路流水线/指令共发，极大提升吞吐量，SIMD 利用CPU宽度，单指令多数据极大提高处理效率
    // “多路流水线/指令共发”指的是现代CPU内部存在多个流水线（能够同时并行处理多条指令的不同阶段）和指令并发执行单元（如多个算数逻辑单元、加载/保存单元）。多路流水线让一次可以有多条指令在不同阶段被同时处理，指令共发（也叫“发射宽度”或“指令并发”）指的是每个时钟周期中可以有多条机器指令被同时派发到不同执行单元。例如，一个“4发射宽度”的处理器可以在一个周期内发出4条指令到相应流水线，大大提升了整体吞吐量和并行计算能力。SIMD向量化进一步扩展这一能力，使单条指令在每条流水线上还可以一次性处理多组数据。
    // 需要什么CPU支持？
    // - 编译器SIMD自动向量化通常针对支持SSE2及以上指令集的CPU（Intel/AMD 2003年之后的x86_64平台基本都支持SSE2）。
    // - OpenMP SIMD指令（#pragma omp simd）本身不要求特殊CPU，但实际是否能自动加速，要依赖编译器和CPU SIMD指令集（SSE/AVX）。
    // - 如果只用#pragma omp simd，加速主要取决于CPU是否支持如SSE、AVX等SIMD指令集，以及你用的编译器是否能实现向量化代码生成。
    // - 建议至少有SSE2支持（现代x86_64必然具备），若有AVX/AVX2/AVX512则向量宽度更大，效果更佳。
    #pragma omp simd  
    for (int i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

### 4.2.2 手动SIMD优化

```cpp
#include <immintrin.h>  // AVX/AVX2

// 使用 AVX2 处理 8 个 int
void add_arrays_avx2(int* a, int* b, int* c, int n) {
    int i = 0;
    // 处理对齐的部分
    for (; i + 7 < n; i += 8) {
        __m256i va = _mm256_load_si256((__m256i*)&a[i]);
        __m256i vb = _mm256_load_si256((__m256i*)&b[i]);
        __m256i vc = _mm256_add_epi32(va, vb);
        _mm256_store_si256((__m256i*)&c[i], vc);
    }
    // 处理剩余元素
    for (; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

## 4.3 CPU缓存优化

### 4.3.1 缓存行大小

```cpp
// 典型的缓存行大小是 64 字节
constexpr size_t CACHE_LINE_SIZE = 64;

// 对齐到缓存行
struct alignas(CACHE_LINE_SIZE) CacheAlignedData {
    int data[16];  // 64 字节
};
```

### 4.3.2 预取（Prefetching）

```cpp
#include <xmmintrin.h>  // SSE

// 手动预取数据到缓存
void process_array(int* arr, int n) {
    for (int i = 0; i < n; ++i) {
        if (i + 1 < n) {
            _mm_prefetch((const char*)&arr[i + 1], _MM_HINT_T0);  // 预取下一个元素
        }
        // 处理 arr[i]
        process(arr[i]);
    }
}

// 使用 __builtin_prefetch
void process_array(int* arr, int n) {
    for (int i = 0; i < n; ++i) {
        if (i + 1 < n) {
            __builtin_prefetch(&arr[i + 1], 0, 3);  // 预取到 L1 缓存
        }
        process(arr[i]);
    }
}
```

## 4.4 指令级并行（ILP）

```cpp
// ❌ 依赖链限制并行
int sum = 0;
for (int i = 0; i < n; ++i) {
    sum += arr[i];  // 每次迭代依赖前一次的结果
}

// ✅ 循环展开增加并行度
int sum = 0;
for (int i = 0; i < n; i += 4) {
    sum += arr[i];
    sum += arr[i + 1];
    sum += arr[i + 2];
    sum += arr[i + 3];
}

// ✅ 更好的方法：多个累加器
int sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;
for (int i = 0; i < n; i += 4) {
    sum0 += arr[i];
    sum1 += arr[i + 1];
    sum2 += arr[i + 2];
    sum3 += arr[i + 3];
}
int sum = sum0 + sum1 + sum2 + sum3;
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-4.-CPU优化.md]
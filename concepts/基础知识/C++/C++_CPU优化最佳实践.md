# C++ CPU优化最佳实践

> 本文总结 C++ 后台服务 CPU 优化的核心实践，涵盖 CPU 亲和性、NUMA 优化、分支预测、SIMD 向量化、频率缩放避免等。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

## 2.1 CPU亲和性与线程绑定

**原理**：将线程绑定到特定 CPU 核心，避免线程迁移带来的缓存失效和上下文切换开销。

```cpp
#include <pthread.h>
#include <sched.h>

// 设置线程CPU亲和性
void set_thread_affinity(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    pthread_t thread = pthread_self();
    pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
}

// C++11方式
#include <thread>
void set_cpu_affinity(std::thread& t, int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    pthread_setaffinity_np(t.native_handle(), sizeof(cpu_set_t), &cpuset);
}
```

**优势**：
- 减少线程迁移开销（避免缓存失效）
- 提高缓存局部性
- 减少上下文切换
- 可预测的性能表现

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

## 2.2 NUMA优化与本地内存分配

**NUMA（Non-Uniform Memory Access）**：多核系统中，不同 CPU 核心访问不同内存区域的速度不同。

```cpp
#include <numa.h>

// 1. 绑定线程到NUMA节点
void bind_to_numa_node(int node) {
    numa_run_on_node(node);      // 线程调度亲和到指定节点
    numa_set_preferred(node);    // 内存分配优先该节点
}

// 2. 从本地NUMA节点分配内存
void* allocate_local_memory(size_t size) {
    int node = numa_node_of_cpu(sched_getcpu());
    return numa_alloc_onnode(size, node);
}

// 3. 检查NUMA拓扑
void print_numa_topology() {
    int max_node = numa_max_node();
    for (int i = 0; i <= max_node; ++i) {
        if (numa_node_size(i, NULL) > 0) {
            printf("Node %d: %ld MB\n", i, numa_node_size(i, NULL) / (1024*1024));
        }
    }
}
```

**numactl工具使用**：
```bash
# 绑定CPU和内存到NUMA节点0
numactl --cpunodebind=0 --membind=0 ./your_program

# 优先使用节点0，不够时自动从其他节点分配
numactl --cpunodebind=0 --mempolicy=preferred,0 ./your_program
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

## 2.3 分支预测优化

**问题**：分支预测失败会导致流水线停顿，增加延迟。

```cpp
// 使用likely/unlikely提示编译器
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

int process(int value) {
    if (likely(value > 0)) {  // 大多数情况下为真
        return value * 2;
    } else {
        return 0;
    }
}

// C++20标准方式
int modern_process(int value, bool is_valid) {
    if (is_valid) [[likely]] {
        return value * 2 + 1;
    } else [[unlikely]] {
        return -1;
    }
}
```

**量化效果**：在极端分支不均衡场景下（如99.9%为true），likely/unlikely可带来10%~40%的延迟减少。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

## 2.4 指令级并行与SIMD向量化

**原理**：让CPU能够同时执行多条指令，提高IPC（每周期指令数）。

```cpp
#include <immintrin.h>

// 向量化累加求和
void vectorized_sum(const int* data, size_t n, int& result) {
    __m256i sum_vec = _mm256_setzero_si256();
    for (size_t i = 0; i < n; i += 8) {
        __m256i vec = _mm256_load_si256((__m256i*)&data[i]);
        sum_vec = _mm256_add_epi32(sum_vec, vec);
    }
    
    int sum[8];
    _mm256_store_si256((__m256i*)sum, sum_vec);
    result = sum[0] + sum[1] + sum[2] + sum[3] + 
             sum[4] + sum[5] + sum[6] + sum[7];
}
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

## 2.5 避免CPU频率缩放

**问题**：CPU频率动态调整会导致性能不稳定。

```bash
# 设置CPU为性能模式（Linux）
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 禁用CPU节能特性
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo  # Intel

# 使用cpupower工具
sudo cpupower frequency-set -g performance
sudo cpupower set -b 0  # 禁用C-states
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

## 2.6 CPU优化 - GitHub项目实践案例

### 2.6.1 DPDK (Data Plane Development Kit) - CPU亲和性
**GitHub**: https://github.com/DPDK/dpdk  
**Stars**: 2.4k+  
**简介**: 用户态数据平面开发工具包，用于高性能网络包处理

**应用场景**：DPDK worker线程绑定到特定CPU核心，避免上下文切换

**关键代码示例**：
```cpp
// dpdk/lib/eal/common/eal_common_thread.c
static int eal_thread_set_affinity(pthread_t thread, const unsigned *lcore_ids) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    for (int i = 0; i < RTE_MAX_LCORE; i++) {
        if (lcore_ids[i] != LCORE_ID_ANY)
            CPU_SET(lcore_ids[i], &cpuset);
    }
    return pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
}
```

**技术说明**：DPDK通过`pthread_setaffinity_np`将数据面处理线程绑定到专用CPU核心，减少缓存失效和上下文切换，实现微秒级延迟。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

### 2.6.2 ClickHouse - SIMD向量化优化
**GitHub**: https://github.com/ClickHouse/ClickHouse  
**Stars**: 32k+  
**简介**: 高性能列式数据库管理系统

**应用场景**：使用SIMD指令加速数据聚合和过滤操作

**关键代码示例**：
```cpp
// ClickHouse/src/Columns/ColumnVector.cpp
template <typename T>
void ColumnVector<T>::applyForAggregates(
    size_t begin, size_t end, AggregateFunctionPtr & func,
    Arena * arena, size_t row_begin, AggregateDataPtr place) const {
    
    // 使用AVX2指令集进行向量化聚合
    if constexpr (sizeof(T) == 8 && std::is_arithmetic_v<T>) {
        if (isArchSupported(TargetArch::AVX2)) {
            // AVX2向量化实现
            __m256i sum_vec = _mm256_setzero_si256();
            const T * data = &data_[begin];
            for (size_t i = 0; i < (end - begin) / 8; ++i) {
                __m256i vec = _mm256_loadu_si256(
                    reinterpret_cast<const __m256i*>(data + i * 8));
                sum_vec = _mm256_add_epi64(sum_vec, vec);
            }
            // 处理剩余元素...
        }
    }
}
```

**技术说明**：ClickHouse使用编译时分支和SIMD指令集检测，在支持AVX2/AVX-512的CPU上自动启用向量化计算，提升数据分析性能5-10倍。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]

### 2.6.3 TensorFlow - 分支预测优化
**GitHub**: https://github.com/tensorflow/tensorflow  
**Stars**: 180k+  
**简介**: 开源机器学习框架

**应用场景**：在热点循环中使用likely/unlikely提示编译器优化分支预测

**关键代码示例**：
```cpp
// tensorflow/core/framework/tensor.cc
bool Tensor::CanUseDMA() const {
    // 大多数情况下张量可以使用DMA
    if (TF_PREDICT_TRUE(dtype() != DT_STRING && dtype() != DT_RESOURCE)) {
        return true;
    }
    // 少数特殊情况需要特殊处理
    return TF_PREDICT_FALSE(IsInitialized()) && 
           TF_PREDICT_FALSE(dtype() == DT_STRING) &&
           TF_PREDICT_FALSE(shape().num_elements() > 0);
}

// tensorflow/core/platform/default/predict.h
#define TF_PREDICT_TRUE(x) (__builtin_expect(!!(x), 1))
#define TF_PREDICT_FALSE(x) (__builtin_expect(x, 0))
```

**技术说明**：TensorFlow在关键路径上使用`TF_PREDICT_TRUE/FALSE`宏提示编译器分支预测，减少流水线停顿，在深度学习推理中提升3-5%性能。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-2.-CPU优化最佳实践.md]
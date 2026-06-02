# CPU 性能优化

## 一、CPU 亲和性与绑定

### 1.1 为什么需要绑定 CPU？

- **避免上下文切换抖动**：线程在不同核心间迁移导致缓存失效
- **NUMA 感知**：线程访问本地内存延迟显著低于远程访问
- **缓存命中率提升**：持续在同一核心运行，L1/L2 缓存保有率高
- **尾延迟（P99/P999）降低**：消除调度不确定性

### 1.2 线程绑定实现

```cpp
#include <pthread.h>
#include <sched.h>

void bind_thread_to_cpu(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    int ret = pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
    if (ret != 0) {
        perror("pthread_setaffinity_np failed");
    }
}

// 多线程示例
void worker_thread(int cpu_id) {
    bind_thread_to_cpu(cpu_id);
    // 业务逻辑...
}

// 生产建议：使用 std::thread 时
#include <thread>
std::thread t([]() {
    bind_thread_to_cpu(0);
    // ...
});
// 注意：需在 t.start() 之后、但在实际业务前调用，或使用 native_handle()
```

### 1.3 taskset 命令行工具

```bash
# 启动时绑定到 CPU 0-3
taskset -c 0-3 ./program

# 绑定已运行的进程
taskset -p -c 0-3 <pid>

# 查看当前绑定
taskset -p <pid>
```

---

## 二、NUMA 优化

### 2.1 NUMA 基础

**NUMA（Non-Uniform Memory Access）**：每个 CPU 插槽有本地内存，访问本地内存比远程内存块。

```bash
# 查看系统 NUMA 拓扑
numactl -H

# 示例输出
available: 2 nodes (0-1)
node 0 cpus: 0-7
node 0 size: 15893 MB
node 1 cpus: 8-15
node 1 size: 16128 MB
node distances:
   node   0   1
     0   10  20    # 跨节点延迟翻倍
     1   20  10
```

**性能差异**：
- 本地访问延迟：80~100ns
- 远程访问延迟：160~200ns（翻倍）
- 带宽也显著下降

### 2.2 NUMA 绑定策略

```cpp
#include <numa.h>
#include <pthread.h>

void bind_thread_and_memory(int cpu_id) {
    // 1. 线程绑定到指定 CPU
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
    
    // 2. 查询 CPU 所属 NUMA 节点
    int node = numa_node_of_cpu(cpu_id);
    
    // 3. 线程调度亲和到该节点
    numa_run_on_node(node);
    
    // 4. 内存分配优先从该节点分配
    numa_set_preferred(node);
}

// 分配指定 NUMA 节点的内存
void* alloc_on_node(size_t size, int cpu_id) {
    int node = numa_node_of_cpu(cpu_id);
    void* ptr = numa_alloc_onnode(size, node);
    if (!ptr) {
        fprintf(stderr, "NUMA 分配失败\n");
        exit(1);
    }
    return ptr;
}

void numa_free_mem(void* ptr, size_t size) {
    numa_free(ptr, size);
}
```

### 2.3 numactl 命令行

```bash
# 严格绑定（节点 0 内存不足时分配失败）
numactl --cpunodebind=0 --membind=0 ./program

# 优先本地分配（不足时自动回退其他节点，推荐）
numactl --cpunodebind=0 --mempolicy=preferred:0 ./program

# 检查进程 NUMA 状态
numactl --show
```

### 2.4 NUMA 感知的内存分配器

- **tcmalloc**：支持 `--tcmalloc_numa_aware` 参数
- **jemalloc**：默认支持 NUMA 感知

---

## 三、缓存优化

### 3.1 缓存行对齐与 False Sharing

**问题**：多线程访问同一缓存行的不同变量，导致缓存行在 CPU 间频繁同步。

```cpp
// 问题代码：False Sharing
struct Counter {
    int count1;  // 线程1频繁修改
    int count2;  // 线程2频繁修改
    // 在同一缓存行（64字节），性能严重下降
};

// 优化：缓存行对齐
struct alignas(64) AlignedCounter {
    int count;
    char padding[64 - sizeof(int)];
};

struct OptimizedCounters {
    AlignedCounter counter1;  // 独占缓存行
    AlignedCounter counter2;  // 独占缓存行
};

// 另一种方式：使用 C++17 std::hardware_destructive_interference_size
#include <new>
struct alignas(std::hardware_destructive_interference_size) Counter {
    int value;
};
```

### 3.2 热数据与冷数据分离

**原理**：将频繁访问的数据紧凑排列，提高缓存命中率。

```cpp
// 优化前：热冷数据混合
struct Entity {
    int id;                    // 热数据：频繁访问
    float position[3];         // 热数据：频繁访问
    std::string description;   // 冷数据：偶尔访问
    std::vector<std::string> tags;  // 冷数据
};

// 优化后：热冷分离
struct EntityHot {
    int id;
    float position[3];
};

struct EntityCold {
    std::string description;
    std::vector<std::string> tags;
};

// 使用 SoA（Structure of Arrays）布局
struct EntitySystem {
    std::vector<int> ids;              // 所有实体的 ID
    std::vector<float> positions_x;    // 所有实体的 X 坐标
    std::vector<float> positions_y;
    std::vector<float> positions_z;
    // 遍历时只有需要的字段被载入缓存
};

// 对比 AoS（Array of Structures）
struct Entity { int id; float x, y, z; };
std::vector<Entity> entities;  // 遍历 id 时会加载不需要的 x, y, z
```

### 3.3 软件预取

```cpp
#include <xmmintrin.h>

void process_array(int* data, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        // 提前预取 8 个元素后的数据到 L1 缓存
        _mm_prefetch((char*)&data[i + 8], _MM_HINT_T0);
        process(data[i]);
    }
}

// 预取提示类型
// _MM_HINT_T0:   L1 缓存（时间局部性）
// _MM_HINT_T1:   L2 缓存（空间局部性）
// _MM_HINT_T2:   L3 缓存
// _MM_HINT_NTA:  非临时访问（不污染缓存）
```

### 3.4 SoA vs AoS 选择

| 场景 | 推荐布局 | 原因 |
|------|----------|------|
| 遍历所有字段 | AoS | 空间局部性好 |
| 只访问部分字段 | SoA | 减少缓存污染 |
| SIMD 处理 | SoA | 便于向量化 |
| 单记录操作 | AoS | 代码简洁 |

---

## 四、CPU 频率与电源管理

### 4.1 设置性能模式

```bash
# 设置 performance 模式，防止 CPU 降频
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 禁用 Turbo Boost（某些场景需要稳定频率）
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# 使用 cpupower 工具
sudo cpupower frequency-set -g performance
sudo cpupower set -b 0  # 禁用 C-states（深度睡眠）
```

### 4.2 CPU 频率缩放对延迟的影响

- **动态调频**：导致性能不稳定，延迟抖动加剧
- **C-States**：深度睡眠后唤醒延迟可达数十微秒
- **适合场景**：
  - 低延迟系统：必须禁用频率缩放
  - 批量处理：可适当开启节能

### 4.3 WSL 环境说明

```bash
# WSL2 不支持 CPU frequency 控制，需在 Windows 宿主设置
# 方法：电源设置 → 选择"高性能"或"卓越性能"电源计划

# .wslconfig 配置示例（%UserProfile%\.wslconfig）
[wsl2]
memory=8GB
processors=4
swap=0                   # 低延迟场景禁用 swap
localhostForwarding=true
```

---

## 五、TLB 与页面管理

### 5.1 大页（Huge Pages）

```bash
# 启用透明大页（THP，慎用）
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# 禁用 THP（低延迟推荐）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 显式分配大页（DPDK 等场景）
echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages

# 挂载大页文件系统
mkdir -p /mnt/huge
mount -t hugetlbfs none /mnt/huge
```

### 5.2 THP 禁用原因（低延迟系统）

- **分配/回收抖动**：THP 的 kcompactd 线程可能在关键路径抢占 CPU
- **内存碎片风险**：大页分配导致碎片
- **延迟不可控**：内核行为不可预知
- **结论**：极致低延迟系统必须禁用 THP

---

## 六、分支与流水线优化

### 6.1 分支预测（已在编译优化中详述，此处列要点）

- 使用 `likely/unlikely` 提示编译器
- 查找表替代 if-else 链
- CMOV 条件移动指令可避免分支

### 6.2 指令级并行

- 循环展开减少数据依赖
- SIMD 向量化单指令多数据
- 避免长延迟指令（如除法）在热点路径

### 6.3 性能指标参考

| 指标 | 正常范围 | 需优化 | 严重问题 |
|------|----------|--------|----------|
| IPC | >1.0 | <0.5 | <0.3 |
| cache-misses | <10% | >25% | >40% |
| branch-misses | <2% | >5% | >10% |
| CPU 利用率 | >90% | <70% | <50%（I/O 瓶颈） |

---

## 七、面试高频追问

### Q1: CPU 亲和性绑定为什么能降延迟？
- 避免线程在不同核心间迁移导致的缓存失效（L1/L2 清空）
- NUMA 架构下保证线程访问本地内存（跨节点延迟翻倍）
- **实测**：绑定 CPU 可降低 P99 延迟 30-50%

### Q2: False Sharing 如何检测？
- **perf stat -e cache-misses** 观察高未命中率
- 加 `alignas(64)` 后对比性能
- Intel VTune / Linux perf 的 cache 事件分析

### Q3: SoA vs AoS 性能差异多大？
- 只访问部分字段时，SoA 减少无效缓存加载
- **实测**：SoA 比 AoS 快 2-5 倍（在只遍历 1-2 个字段的场景）
- SIMD 向量化必须 SoA

### Q4: THP 为什么对低延迟系统有害？
- THP 的 kcompactd 线程可能在关键路径上抢占 CPU
- 大页分配/回收不可控，导致延迟抖动
- **结论**：极致低延迟系统必须 `echo never > /transparent_hugepage/enabled`

### Q5: Linux 下如何确认 CPU 降频导致性能问题？
```bash
# 查看当前频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 查看可用频率范围
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# 设置为 performance 模式
sudo cpupower frequency-set -g performance
```

# NUMA 架构

See also: [[内存管理]], [[性能优化]]

## 一、概述

NUMA (Non-Uniform Memory Access) 是一种多处理器计算机的内存架构，每个 CPU 有自己的本地内存，访问本地内存比访问远程内存快。

## 二、核心知识点

### 1. NUMA 拓扑

```
┌─────────────────────┐    ┌─────────────────────┐
│     Node 0          │    │     Node 1          │
│  ┌───────────────┐  │    │  ┌───────────────┐  │
│  │   CPU 0-7     │  │    │  │   CPU 8-15    │  │
│  └───────┬───────┘  │    │  └───────┬───────┘  │
│          │          │    │          │          │
│  ┌───────┴───────┐  │    │  ┌───────┴───────┐  │
│  │  本地内存 0   │◄─┼────┼─►│  本地内存 1   │  │
│  │   (快)       │  │ 互连 │  │   (快)       │  │
│  └───────────────┘  │    │  └───────────────┘  │
└─────────────────────┘    └─────────────────────┘
```

- **本地访问**：CPU 访问自己 Node 的内存，延迟低
- **远程访问**：CPU 访问其他 Node 的内存，延迟高（1.5-2x）

### 2. NUMA 感知编程

```cpp
#include <numa.h>

// 查看 NUMA 拓扑
int num_nodes = numa_max_node() + 1;

// 在指定节点分配内存
void* ptr = numa_alloc_onnode(size, node_id);

// 绑定线程到指定 NUMA 节点
struct bitmask* mask = numa_allocate_nodemask();
numa_bitmask_setbit(mask, node_id);
numa_bind(mask);
numa_free_nodemask(mask);

// 释放 NUMA 分配的内存
numa_free(ptr, size);
```

### 3. 查看 NUMA 信息

```bash
# 查看 NUMA 拓扑
numactl --hardware

# 查看每个节点的内存信息
numastat

# 在指定节点运行程序
numactl --cpunodebind=0 --membind=0 ./program
```

### 4. 内存分配策略

| 策略 | 含义 |
|------|------|
| `MPOL_DEFAULT` | 默认，按进程所在节点分配 |
| `MPOL_BIND` | 严格绑定到指定节点 |
| `MPOL_INTERLEAVE` | 轮询分配到多个节点 |
| `MPOL_PREFERRED` | 优先指定节点，可回退 |

```cpp
#include <numaif.h>

// 设置内存策略
unsigned long nodemask = 1 << node_id;
set_mempolicy(MPOL_BIND, &nodemask, sizeof(nodemask) * 8);
```

### 5. 自定义分配器

```cpp
#include <memory>

template<typename T>
struct NumaAllocator {
    using value_type = T;
    int node_id;
    
    NumaAllocator(int node = 0) : node_id(node) {}
    
    T* allocate(std::size_t n) {
        void* p = numa_alloc_onnode(n * sizeof(T), node_id);
        return static_cast<T*>(p);
    }
    
    void deallocate(T* p, std::size_t n) {
        numa_free(p, n * sizeof(T));
    }
};

// 使用
std::vector<int, NumaAllocator<int>> v(NumaAllocator<int>(0));
```

## 三、面试重点

1. **内存对齐**：`alignas` / `aligned_alloc`
2. **NUMA架构**：多CPU插槽服务器的内存访问
3. **内存碎片**：内碎片 vs 外碎片

[src: raw/ingested/2技术/cpp/C++性能优化代码复习指南.md]

## Related Pages
- [[内存管理]]
- [[性能优化]]
- [[C++多线程与并发]]
- [[Linux线程调度]]
- [[Intel_TBB]]
- [[C++并行算法]]

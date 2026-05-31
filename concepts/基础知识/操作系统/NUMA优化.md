# NUMA 优化

**NUMA（Non-Uniform Memory Access）**：多核系统中，不同 CPU 核心访问不同内存区域的速度不同。

## NUMA 是什么？

NUMA（Non-Uniform Memory Access，非一致性内存访问）是一种多处理器系统的内存架构。在 NUMA 系统中，处理器被划分为不同的节点，每个节点有自己的本地内存。处理器访问本地内存比访问其他节点的远程内存速度更快、延迟更低。

- **举例说明**：在一个有两个 CPU 的服务器上，每个 CPU 都有自己的内存条。如果 CPU0 访问自己的内存很快，但访问 CPU1 的内存就比较慢，这就是 NUMA 架构。

- **NUMA 和 CPU 缓存的关系？**  
  NUMA 架构描述的是“内存访问延迟”，而不是 CPU 缓存；NUMA 指的是不同 CPU 访问本地或远程“内存条”的快慢。CPU 缓存（如 L1/L2/L3）主要是为加速 CPU 和内存之间的数据传输，和 NUMA 属于不同层级。NUMA 优化的核心是让线程优先访问本地内存以减少远程访问延迟，不是针对 CPU 缓存的优化。

## 怎么知道 CPU 跟哪个内存接近？

在 NUMA 架构下，CPU 和内存的拓扑关系通常可以用“NUMA 节点”描述。每个 NUMA 节点包含一组 CPU 核心和对应的本地内存。判断 CPU 与哪个内存接近，实际上就是要知道“某个 CPU 属于哪个 NUMA 节点”。

**常用方法：**

1. **使用 lscpu 或 numactl 工具（Linux）**
    - 运行 `lscpu` 可以直接看到每个 NUMA 节点包含的 CPU 列表。
    - 运行 `numactl --hardware`（或 `numactl -H`）可以列出 NUMA 拓扑，包括每个节点的 CPU 和内存。
    - 例子：
      ```
      $ lscpu
      ...
      NUMA node0 CPU(s):     0-7
      NUMA node1 CPU(s):     8-15
      ```

2. **代码查询 NUMA 拓扑**
    - 可以用 `numa_node_of_cpu(int cpu)` 查询某个 CPU 属于哪个 NUMA 节点。
    - 例子（伪代码）：
      ```cpp
      #include <numa.h>
      int cpu = sched_getcpu();
      int node = numa_node_of_cpu(cpu);
      printf("CPU %d 属于 NUMA 节点 %d\n", cpu, node);
      ```

3. **/sys 文件系统**
    - Linux 下 `/sys/devices/system/node/` 目录下有每个 NUMA 节点的 cpu 列表。

## 为什么要做 NUMA 优化？

由于 NUMA 架构下本地和远程内存访问的性能差异较大，如果线程经常跨节点访问远程内存，会导致显著的性能损失，包括更高的延迟和更低的带宽。

- **NUMA 优化的目的**：
    - 让线程优先访问本地内存，减少跨节点访问，最大化内存带宽和最小化访问延迟；
    - 在高并发、低延迟系统中，充分利用 NUMA 架构带来的优势，提升整体性能和可扩展性。

- **通常优化措施**：
    - 让线程运行在本地 NUMA 节点上，并优先分配本节点的内存资源；
    - 使用如 `numactl` 或编程接口，将内存和计算亲和在同一个 NUMA 节点上；
    - 数据结构尽量局部化，避免不必要的跨节点访问。

## 优化策略

```cpp
#include <numa.h>

// 1. 绑定线程到 NUMA 节点
void bind_to_numa_node(int node) {
    numa_run_on_node(node);
    numa_set_preferred(node);
}

// 2. 从本地 NUMA 节点分配内存
void* allocate_local_memory(size_t size) {
    int node = numa_node_of_cpu(sched_getcpu());
    return numa_alloc_onnode(size, node);
}

// 3. 检查 NUMA 拓扑
void print_numa_topology() {
    int max_node = numa_max_node();
    for (int i = 0; i <= max_node; ++i) {
        if (numa_node_size(i, NULL) > 0) {
            printf("Node %d: %ld MB\n", i, numa_node_size(i, NULL) / (1024*1024));
        }
    }
}
```

## 最佳实践

- 线程和其使用的数据应在同一 NUMA 节点
- 避免跨 NUMA 节点访问内存
- 使用 `numactl` 工具启动程序

[src: raw/ingested/2技术/性能优化/低延迟-低延迟c++系统分析-二、CPU-优化技术-二、CPU-优化技术.md]
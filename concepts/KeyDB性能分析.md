# KeyDB 性能分析

> 本文分析 KeyDB 高性能的六大原因，包括对称多线程事件循环、SO_REUSEPORT、线程绑核、FastLock 自旋锁、极短锁持有时间和 Async Rehash。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

## 四、为什么 KeyDB 性能这么高

### 4.1 原因一：对称多线程事件循环

Redis 的单线程模型意味着无论有多少 CPU 核心，单实例只能利用一个核心。KeyDB 的每个 Worker 线程都运行完整的事件循环：

```
Worker Thread 生命周期：
  1. epoll_wait() 等待网络事件
  2. accept() 新连接（SO_REUSEPORT 多线程共享端口）
  3. read() 客户端请求数据
  4. 解析 RESP 协议
  5. 获取 FastLock
  6. 执行命令，操作内存数据结构
  7. 释放 FastLock
  8. write() 响应数据回客户端
```

步骤 1~4 和步骤 8 完全并行无锁。**只有步骤 5~7 需要短暂持锁**。这意味着大部分时间（网络 I/O 和协议解析）是完全并行的。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

### 4.2 原因二：SO_REUSEPORT 消除 accept 瓶颈

传统模型中，多个线程竞争同一个 listen socket 的 accept() 调用会产生**惊群效应（thundering herd）**。KeyDB 使用 Linux `SO_REUSEPORT` 特性：

- 允许多个线程各自 bind + listen 同一端口
- 内核在 TCP 层面做负载均衡，将新连接分配给不同线程
- 配合 `SO_INCOMING_CPU` 将数据导向对应 CPU 核心
- 消除了传统 accept 竞争瓶颈

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

### 4.3 原因三：线程绑核（CPU Affinity）

KeyDB 支持 `--server-thread-affinity true` 参数，将每个 Worker 线程绑定到特定 CPU 核心：

- 提升 CPU L1/L2 缓存命中率
- 减少线程在核心间迁移的开销
- 减少缓存行失效（cache line invalidation）
- 配合 `SO_INCOMING_CPU` 实现数据亲和性

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

### 4.4 原因四：FastLock 自旋锁而非 mutex

KeyDB 没有使用 `pthread_mutex`，而是自研了一个基于 **x86-64 汇编** 的 FastLock 自旋锁：

```
// fastlock 结构（简化）
struct fastlock {
    volatile int32_t m_pidOwner;  // 持锁线程 ID
    volatile int32_t m_depth;     // 递归深度
    volatile uint16_t active;     // 当前活跃令牌
    volatile uint16_t avail;      // 下一个可用令牌
};
```

**为什么用汇编实现而非 C？** 开发团队发现 GCC 对自旋锁的 C 代码生成了次优的机器码。自旋锁是 KeyDB 最关键的性能热点路径，使用汇编可以精确控制指令序列，确保最优的原子操作和内存序语义。

**FastLock vs pthread_mutex 的优势：**

| 特性 | pthread_mutex | FastLock |
|------|--------------|----------|
| 等待方式 | 系统调用 futex，线程休眠 | 自旋忙等（spin） |
| 上下文切换 | 需要内核态切换 | 纯用户态 |
| 获锁延迟 | 微秒级（含调度延迟） | 纳秒级 |
| 适用场景 | 长时间持锁 | **短时间持锁**（正是 KeyDB 的场景） |
| 递归支持 | 需要 RECURSIVE 属性 | 原生支持（m_depth 字段） |

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

### 4.5 原因五：锁持有时间极短

这是 KeyDB 高并行度的**核心关键**。

Redis 的单次命令执行中，时间分布大致如下：

```
┌─────────────────────────────────────────────┐
│          一次请求的时间分布                    │
├─────────────┬──────────┬────────────────────┤
│ 网络 I/O    │协议解析   │ 数据结构操作        │
│ (~60-70%)   │ (~15-20%)│ (~10-20%)          │
├─────────────┴──────────┴────────────────────┤
│  ←── 无锁并行 ──→  ←── 无锁 ──→ ←─ 持锁 ─→ │
└─────────────────────────────────────────────┘
```

KeyDB 的哈希表（dict）操作是 O(1) 的，对于 GET/SET 等简单命令，实际在持锁状态下的操作**只涉及几次内存访问**（纳秒级别）。这意味着：

- 锁竞争（lock contention）极低
- 自旋等待时间极短
- **大部分 CPU 时间花在无锁的网络 I/O 和协议解析上**
- 因此多线程的 CPU 利用率非常高，极少浪费在等锁上

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

### 4.6 原因六：Async Rehash——自旋等待时间利用

传统的自旋锁在等待时浪费 CPU 做空循环。KeyDB v6.3 引入了 **Async Rehash** 机制：

- 当线程在自旋等待 FastLock 时，利用这些 CPU 周期执行 **hash table rehash**
- 将原本浪费的自旋时间转化为有用的后台工作
- 在很多场景下几乎完全隐藏了 rehash 的开销

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-四、为什么-KeyDB-性能这么高.md]

## Related Pages
- [[KeyDB存算分离项目]]
- [[Redis性能问题]]
- [[C++多线程与并发]]
- [[C++并发性能优化]]
- [[C++内联汇编]]
- [[线程同步机制]]
- [[POSIX线程管理]]
# KeyDB 性能测试数据

> 本文汇总 KeyDB 官方及第三方性能测试数据，涵盖吞吐量、延迟、多线程扩展性等关键指标。

See also: [[KeyDB存算分离项目]], [[Redis存储型方案性能对比]], [[Redis性能问题]]

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

---

## 一、KeyDB 官方基准测试

### 测试环境

| 项目 | 配置 |
|------|------|
| **KeyDB 实例机器** | AWS m5.4xlarge（16 vCPU, 64GB RAM） |
| **压测客户端机器** | AWS m5.8xlarge（32 vCPU, 128GB RAM） |
| **压测工具** | Memtier（RedisLabs 出品），32 线程 |
| **KeyDB 启动参数** | `--server-threads 7 --server-thread-affinity true` |
| **Redis 启动参数** | 默认单线程 |

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

### 官方核心结论

- 单实例 KeyDB 的 ops/sec 是 Redis v5 的 **5.13x ~ 5.49x**
- 延迟降低至 Redis 的 **1/5**（约 4.6x ~ 5.1x 改善）
- 单个 KeyDB 节点的吞吐量 **等于 7 节点 Redis 集群**

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

### 线程数 vs 性能线性扩展

官方测试显示，随着 `--server-threads` 增加，ops/sec **近乎线性增长**：

| 线程数 | 大致 ops/sec（SET+GET 混合） | 相对 1 线程倍数 |
|--------|-------------------------------|-----------------|
| 1 | ~200K | 1x |
| 2 | ~400K | ~2x |
| 3 | ~550K | ~2.75x |
| 4 | ~700K | ~3.5x |
| 5 | ~830K | ~4.15x |
| 6 | ~920K | ~4.6x |
| 7 | ~1,000K | ~5x |

> 说明：1 线程的 KeyDB 仍比同配置下的 Redis 快约 5%，因为 KeyDB 在代码层面也有其他优化。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

---

## 二、OpenBenchmarking.org 基准测试

测试配置：KeyDB 6.3.2，SET 操作，50 并行连接。

| 硬件平台 | 核心/线程 | QPS（req/sec） |
|----------|-----------|----------------|
| Intel Core i9-13900K | 24C/32T | 2,036,729 ±31K |
| AMD Ryzen 7 PRO 5850U | 8C/16T | 1,955,627 ±95K |
| AMD Ryzen 9 3900XT | 12C/24T | 1,527,303 ±2.4K |
| Intel Core i5-12400 | 6C/12T | 1,308,405 ±98K |
| Intel Core i7-1185G7 | 4C/8T | 1,004,346 ±30K |
| Intel Core i7-1065G7 | 4C/8T | 792,065 ±8.5K |

中位数性能：**~1,253,730 QPS**。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

---

## 三、KeyDB 官方 vs Redis 6 vs Elasticache（AWS r5 系列实例）

### Memtier 吞吐量对比

官方在 AWS r5 系列实例上使用 Memtier 32 线程进行测试，KeyDB 使用 `--server-threads 7 --server-thread-affinity true`，Redis 6 使用 `--io-threads 4~8`：

| 场景 | Redis 6 | Elasticache | KeyDB |
|------|---------|-------------|-------|
| r5.large (2 vCPU) | 基准 | 略高于 Redis | 高于 Redis ~50% |
| r5.xlarge (4 vCPU) | 基准 | 略低于 r5.large（异常） | 明显领先 |
| r5.2xlarge (8 vCPU) | 基准 | 高于 Redis | 远超 Redis |
| r5.4xlarge (16 vCPU) | 基准 | 高于 Redis | **大幅领先**（~5x） |

> 核心发现：随着可用 CPU 核心增多，KeyDB 的扩展优势越来越明显，而 Redis 6 的 io-threads 超过 4 个后通常不再有提升。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

### YCSB 延迟对比（Workload A: 50/50 读写）

测试在 r5.4xlarge 上进行，350 客户端线程，1000 万记录：

| 目标吞吐量 | Redis 6 延迟(μs) | Elasticache 延迟(μs) | KeyDB 延迟(μs) |
|------------|-------------------|----------------------|----------------|
| 低负载 | 相近 | 相近 | 相近 |
| 中等负载 | 相近 | 相近 | 相近 |
| 接近容量上限 | **急剧上升** | 中等上升 | **保持低延迟** |

> 关键结论：三者在低中负载下延迟接近，但在高负载时 KeyDB 因容量天花板更高，延迟更优。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

---

## 四、KeyDB vs Redis（高并发场景对比）

| 操作类型 | Redis（ops/sec） | KeyDB（ops/sec） | 提升幅度 |
|----------|------------------|-------------------|----------|
| SET（100+ 连接） | 150,000 | 450,000 | +200% |
| GET（100+ 连接） | 180,000 | 550,000 | +206% |
| LPUSH（100+ 连接） | 140,000 | 420,000 | +200% |
| SET（Pipeline 10） | 800,000 | 1,800,000 | +125% |
| GET（Pipeline 10） | 950,000 | 2,100,000 | +121% |
| 单连接顺序请求 | 基准 | +5~6% | 微小提升 |

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

---

## 五、第三方独立测试（InfraCloud）

InfraCloud 进行了大规模独立测试。

### 硬件配置

| 项目 | 配置 |
|------|------|
| CPU | 112 核 |
| 内存 | 1 TB RAM |
| 网络 | 50 Gbps |
| 磁盘 | 1 TB |
| 服务器数量 | 3 台 |
| Redis 版本 | 7.0.11（多线程模式，io-threads 20） |
| KeyDB 版本 | 6.3.3（server-threads 10） |

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

### 测试结果

**性能测试（600 万请求，2MB 数据包）**

| 指标 | Redis 多线程 | KeyDB |
|------|-------------|-------|
| 总处理请求/秒 | 11,093 | 3,778 |
| SET ops/sec | 1,089 | 343 |
| GET ops/sec | 10,083 | 3,434 |
| 总耗时 | 549s | 1,603s |
| p99.9 延迟 | 1,032ms | 864ms |

**复制测试（5.47 亿条记录加载）**

| 指标 | Redis 多线程 | KeyDB |
|------|-------------|-------|
| 总耗时 | 14 分钟 | 18 分钟 |
| SET 调用/秒 | ~750K | ~500K |

> [!contradiction]
> New source claims KeyDB performance is significantly lower than Redis in InfraCloud tests, but previous sources (KeyDB official benchmarks) claim KeyDB is 5x faster than Redis. The discrepancy is explained by the source itself: InfraCloud's test used 2MB payload for KeyDB vs 2KB for Redis, making it an unfair comparison. Additionally, the test only hit a single KeyDB master node, not utilizing the 3-node Active-Replica advantage.

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-二、性能测试数据汇总.md]

---

## Related Pages
- [[KeyDB存算分离项目]]
- [[Redis存储型方案性能对比]]
- [[Redis性能问题]]
- [[RocksDB文件体系]]
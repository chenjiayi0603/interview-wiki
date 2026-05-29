# RocksDB 版本演进

RocksDB 从版本 6 到 8（尤其是 7.x 到 8.x 的演进）在性能、稳定性和功能扩展上实现了显著提升。

## 一、版本特性对比

| 特性 | RocksDB 6.x | RocksDB 7.x+ | RocksDB 8.x（预期） |
|------|------------|-------------|-------------------|
| Compaction 稳定性 | 高负载下易阻塞写入 | 写入 QPS 波动 < 5% | 进一步优化后台任务调度 |
| 多线程支持 | 基础并行 Compaction | 细粒度锁控制 + 增量 Compaction | 自适应并发策略 |
| 内存管理 | LRU Cache 为主 | Partitioned Index + jemalloc 集成 | 智能内存分层分配 |
| 压缩算法 | Snappy/Zlib/LZ4 | 新增 Zstd，支持分层压缩 | 算法动态调优 |
| 生态兼容性 | C++11 基础支持 | 要求 C++17+，提升现代硬件利用率 | 强化云原生集成 |

## 二、RocksDB 7.x 核心优化

### 1. Compaction 阻塞问题根治

- **问题背景**：6.x 版本在持续高写入负载下，Compaction 可能导致写入完全停滞（Write Stall），QPS 骤降为 0
- **优化方案**：
  - 修复 Pending Bytes 计算放大问题（Issue #9423），避免过度触发 Write Stall
  - 减少 Compaction 与写入的锁竞争，通过延长 Compaction 周期、降低 I/O 突发压力
- **效果**：7.5.3+ 版本在 Compaction 期间写入延迟波动极小，QPS 稳定性提升 10 倍以上

### 2. 多线程与并发控制增强

- 支持多线程并行 Compaction，通过 `subcompactions` 参数拆分任务
- 优化全局 Mutex 锁粒度，减少线程竞争（如 ClockCache 替代 LRU Cache）
- 增量 Compaction：分批次处理数据，降低单次资源占用

### 3. 内存分配与索引优化

- **jemalloc 集成**：替代默认内存分配器，减少内存碎片，提升多线程性能
- **分区索引（Partitioned Index/Filter）**：引入多级索引结构，将索引顶层常驻内存，下层按需加载，减少 Swap 竞争

### 4. 压缩算法扩展

- 新增 Zstandard（Zstd）算法，兼顾高压缩率与速度（尤其适合冷数据）
- 动态压缩策略：可分层设置压缩算法（如 L0 禁用压缩、底层启用 Zstd）

## 三、生产环境版本选择

### 首选推荐：RocksDB 7.x（尤其是 7.5.3+）

- Compaction 性能突破：Compaction 期间写入 QPS 波动 < 5%
- 内存管理增强：分区索引在内存受限场景性能提升可达 10 倍
- 生产验证广泛：Kvrocks、MyRocks 等知名项目已升级至 7.x

### 保守选择：RocksDB 6.29.5（长期稳定版）

- 作为 6.x 的最终版本，修复了早期关键 Bug
- 编译依赖要求低（C++11），兼容旧硬件和操作系统
- 适合嵌入式设备或低资源环境

## 四、升级注意事项

- **环境依赖**：7.x 需 C++17 编译环境（GCC 7+），旧系统升级可能需更新工具链
- **关键参数调优**：
  - 启用分区索引：`state.backend.rocksdb.memory.partitioned-index-filters=true`
  - 限制 Compaction I/O 突发：`soft_pending_compaction_bytes_limit=64GB`
- **验证与监控**：使用 `db_bench` 压测 Compaction 期间写入稳定性

[src: raw/ingested/2技术/rocksdb/rocksdb.md]

## Related Pages
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]
- [[RocksDB LSM-Tree]]

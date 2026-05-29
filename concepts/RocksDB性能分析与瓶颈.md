# RocksDB 性能分析与瓶颈

本文档聚焦 RocksDB 的**性能分析**、**瓶颈根因**与**优化手段**，便于面试复习和线上排查。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

## 一、性能分析维度

### 1.1 核心指标

| 指标类别 | 关键指标 | 说明 |
|----------|----------|------|
| **吞吐** | 读/写 QPS | 每秒查询数、写入数 |
| **延迟** | P50/P95/P99 | 分位延迟，关注 P99 |
| **IO** | IOPS、带宽 | 磁盘实际读写量 |
| **资源** | CPU、内存、磁盘空间 | 资源占用与饱和度 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 1.2 三大放大指标

| 放大类型 | 含义 | 观测方式 |
|----------|------|----------|
| **写放大** | 逻辑写入 vs 物理写入 | Compaction IO / 用户写入量 |
| **读放大** | 逻辑读取 vs 物理 IO | 单次 Get 可能读多个 SST |
| **空间放大** | 磁盘占用 vs 逻辑数据量 | 磁盘使用量 / 实际数据量 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 1.3 常用观测接口

```cpp
// 获取统计信息
std::string value;
db->GetProperty("rocksdb.stats", &value);           // 全局统计
db->GetProperty("rocksdb.num-files-at-level0", &value);  // L0 文件数
db->GetProperty("rocksdb.compaction-pending", &value);   // 是否有待 Compaction
db->GetProperty("rocksdb.is-write-stopped", &value);     // 是否 Write Stall
db->GetProperty("rocksdb.num-immutable-mem-table", &value);  // Immutable MemTable 数
db->GetProperty("rocksdb.block-cache-hit-rate", &value);     // Block Cache 命中率
```

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 1.4 性能参考量级（单机 SSD）

| 场景 | 读 QPS 量级 | 写 QPS 量级 | 说明 |
|------|-------------|-------------|------|
| 高 Cache 命中 + sync=false（稳定、持续写入下） | 18~25 万   | 18~25 万   | 大部分请求命中 Block Cache，仅内存访问，极高吞吐，但测试为长时间稳定运行下非瞬时峰值 |
| 读需频繁落盘 + sync=false                    | 1~5 万     | 18~25 万   | 读操作需要较多磁盘 IO，受 SSD 能力限制，写仍主要是 WAL 内存写入，高并发下波动较小 |
| sync=true（每次 fsync）                      | —         | 7~10 万    | 每次写都强制 fsync，受 SSD 持续写入能力影响，写延迟升高，QPS 下降，读性能影响不大 |
| HDD                                          | 数千~1.5 万 | 1~2 万     | 机械硬盘 IO 延迟大，稳定吞吐量受限，远低于 SSD 场景 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

---

## 二、瓶颈原因分析

### 2.1 写入瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **WAL sync** | `sync_log=true` 时每次写都 fsync | 写延迟高、QPS 低 |
| **Immutable MemTable 堆积** | Flush 跟不上，超过 `max_write_buffer_number` | Write Stall |
| **L0 文件过多** | Compaction 跟不上，超过 `level0_slowdown_writes_trigger` | Write Slowdown |
| **L0 文件过多** | Compaction 严重滞后，超过 `level0_stop_writes_trigger` | Write Stop |
| **Pending Compaction 过大** | 待合并数据量超过 `soft/hard_pending_compaction_bytes_limit` | 写入限流 |
| **单线程 WAL** | WAL 写入串行 | 高并发下成为瓶颈 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 2.2 读取瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **L0 文件多** | L0 无序、可重叠，需遍历多个 SST | 点查延迟高 |
| **Block Cache 命中低** | Cache 太小或热点分布变化 | 大量磁盘 IO |
| **无 Bloom Filter** | 无效 SST 也被读入 | 读放大严重 |
| **层级深** | 每层可能查一个 SST，层数多则 IO 多 | 读路径长 |
| **Index/Filter 未缓存** | 每次查 SST 都要读 Index/Filter | 额外 IO |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 2.3 Compaction 相关瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **Compaction 线程不足** | `max_background_compactions` 过小 | L0 堆积、Write Stall |
| **写放大高** | Leveled Compaction 导致多次重写 | 磁盘 IO 压力大 |
| **Compaction 与前台争抢 IO** | 无 Rate Limiter 或限流过松 | 前台延迟抖动 |
| **单次 Compaction 过大** | 未开启 `max_subcompactions` | CPU 利用率低、耗时长 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 2.4 资源瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **内存不足** | Block Cache + MemTable + 索引占满 | 频繁换页、性能下降 |
| **磁盘 IOPS 不足** | 写放大 + 读放大叠加 | 延迟抖动、QPS 下降 |
| **CPU 压缩** | 压缩算法过重（如 Zlib） | CPU 成为瓶颈 |
| **磁盘空间** | Compaction 临时空间 + 多版本数据 | 空间不足或 OOM |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

---

## 三、优化手段

### 3.1 写入优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **WAL sync 策略** | 非强一致场景下 `sync_log=false` | 显著降低写延迟 |
| **批量写入** | 使用 WriteBatch 合并多次写入 | 减少 WAL 和锁竞争 |
| **并发 MemTable 写** | 多线程并发写 MemTable | `allow_concurrent_memtable_write=true` |
| **流水线写入** | WAL 与 MemTable 写入流水线化 | `enable_pipelined_write=true` |
| **增大 MemTable** | 减少 Flush 频率 | `write_buffer_size=256MB` |
| **增加 Flush 线程** | 加快 Immutable 落盘 | `max_background_flushes=2~4` |
| **增加 Compaction 线程** | 控制 L0、避免 Write Stall | `max_background_compactions=4~8` |
| **子 Compaction** | 单次 Compaction 并行化 | `max_subcompactions=4` |
| **Universal Compaction** | 降低写放大 | 写密集型场景 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 3.2 读取优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **Block Cache** | 缓存 Data/Index/Filter Block | 建议总内存 1/3 |
| **Index/Filter 缓存** | 避免每次查 SST 都读元数据 | `cache_index_and_filter_blocks=true` |
| **L0 索引常驻** | L0 索引和 Filter 常驻缓存 | `pin_l0_filter_and_index_blocks_in_cache=true` |
| **Bloom Filter** | 快速排除不可能存在的 Key | `NewBloomFilterPolicy(10)` |
| **分区索引** | 7.x+ 降低 Swap 竞争 | `partitioned_index_filters=true` |
| **控制 L0 文件数** | Compaction 及时，减少 L0 | 调大 `level0_file_num_compaction_trigger` 需谨慎 |
| **增大 Block Size** | 顺序读友好 | `block_size=16KB~64KB` |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 3.3 Compaction 优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **动态层级** | 按最深层动态调整各层大小 | `level_compaction_dynamic_level_bytes=true` |
| **优先级策略** | 优先合并重叠小的文件 | `compaction_pri=kMinOverlappingRatio` |
| **Rate Limiter** | 限制 Compaction IO，避免抢前台 | `NewGenericRateLimiter(100MB/s)` |
| **Pending 阈值** | 合理设置避免过度 Write Stall | `soft/hard_pending_compaction_bytes_limit` |
| **策略选择** | 读多→Leveled，写多→Universal | 按负载选择 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 3.4 压缩与资源优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **L0 不压缩** | 减少 CPU | `compression=kNoCompression` |
| **底层 Zstd** | 冷数据高压缩率 | `bottommost_compression=kZstd` |
| **中间层 LZ4** | 平衡速度与压缩率 | `compression=kLZ4` |
| **bytes_per_sync** | 减少同步次数 | `bytes_per_sync=1MB` |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 3.5 场景化配置速览

```cpp
// 写密集型
options.compaction_style = kCompactionStyleUniversal;
cf_options.write_buffer_size = 256 << 20;
cf_options.max_write_buffer_number = 4;
options.max_background_jobs = 12;
options.max_subcompactions = 8;

// 读密集型
options.compaction_style = kCompactionStyleLevel;
table_options.block_cache = NewLRUCache(2ULL << 30);  // 2GB
table_options.cache_index_and_filter_blocks = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;

// 内存受限
table_options.partitioned_index_filters = true;
cf_options.write_buffer_size = 16 << 20;
cf_options.max_write_buffer_number = 2;

// 低延迟 / 避免 Write Stall
options.level0_slowdown_writes_trigger = 99999;  // 谨慎使用
options.level0_stop_writes_trigger = 99999;
options.rate_limiter.reset(NewGenericRateLimiter(100 << 20));
```

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

---

## 四、排查与调优流程

### 4.1 写入变慢排查

1. **是否 Write Stall**：`is-write-stopped`、`num-immutable-mem-table`
2. **L0 是否堆积**：`num-files-at-level0`
3. **Compaction 是否滞后**：`compaction-pending`、`pending-compaction-bytes`
4. **调整方向**：增加 Flush/Compaction 线程、调整 Compaction 策略、可选关闭/放宽 sync

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 4.2 读取变慢排查

1. **Block Cache 命中率**：`block-cache-hit-rate`（建议 > 90%）
2. **L0 文件数**：`num-files-at-level0`
3. **是否启用 Bloom Filter**：检查 `table_factory` 配置
4. **调整方向**：增大 Block Cache、启用 Index/Filter 缓存、优化 Bloom Filter

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### 4.3 监控清单

| 指标 | 关注点 |
|------|--------|
| 写延迟 P99 | 是否明显升高 |
| Write Stall 次数/时长 | 是否频繁触发 |
| L0 文件数 | 是否持续偏高 |
| Pending Compaction Bytes | 是否逼近阈值 |
| Block Cache 命中率 | 是否低于 90% |
| 磁盘 IOPS/带宽 | 是否饱和 |
| CPU 使用率 | 压缩是否成为瓶颈 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

## Related Pages
- [[RocksDB读取流程]]
- [[RocksDB写入流程]]
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]
- [[RocksDB LSM-Tree]]
- [[OBS对接RocksDB性能分析]]

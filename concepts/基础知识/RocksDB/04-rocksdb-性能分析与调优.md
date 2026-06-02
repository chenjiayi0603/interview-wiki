# RocksDB 性能分析与调优

> 三大放大、性能基准、瓶颈原因与优化手段、监控排查。

---

## 一、三大放大问题

### 1.1 写放大（Write Amplification）

**定义**：物理写入数据量 / 逻辑写入数据量

**原因**：
| 阶段 | 放大倍数 | 说明 |
|------|---------|------|
| WAL 写入 | 1x | 写 WAL |
| MemTable 刷盘 | 1x | 写 L0 SST |
| L0→L1 Compaction | 2-3x | 读取 L0 + L1，写入 L1 |
| L1→L2 Compaction | 2-3x | 读取 L1 + L2，写入 L2 |
| 总计 | **5-10x** | 取决于层级深度 |

**优化策略**：
| 手段 | 效果 |
|------|------|
| Universal Compaction | 写放大降至 2-3x |
| 增大 MemTable（`write_buffer_size`） | 减少 Flush 频率 |
| 调整触发条件（`level0_file_num_compaction_trigger`） | 减少 Compaction 频率 |
| 启用子 Compaction（`max_subcompactions`） | 缩短单次 Compaction 时间 |

### 1.2 读放大（Read Amplification）

**定义**：读取数据量 / 实际数据量

**原因**：
- L0 需要遍历所有文件（无序，可能重叠）
- 多层级查询（MemTable → L0 → L1 → ... → Ln）
- 每个 SST 内需要从 Data Block 中查找

**优化策略**：
| 手段 | 效果 |
|------|------|
| Bloom Filter（`bits_per_key=10`） | 快速过滤不存在 Key，减少 90%+ 无效 IO |
| Block Cache（建议总内存 1/3） | 缓存热数据块，减少磁盘读取 |
| 分区索引（7.x+ `partitioned_index_filters`） | 减少 Swap 竞争 |
| 控制 L0 文件数 | 及时 Compaction，减少遍历 |

### 1.3 空间放大（Space Amplification）

**定义**：磁盘占用 / 实际数据大小

**原因**：
- 同一 Key 的多个版本同时存在
- 删除标记（Tombstone）未及时清理
- Compaction 不及时

**优化策略**：
| 手段 | 效果 |
|------|------|
| Leveled Compaction | 每层 Key 唯一，空间放大低 |
| 及时 Compaction | 清理 Tombstone |
| 动态层级大小（7.x+） | 空间效率更稳定 |

### 1.4 三大放大的权衡

| 放大类型 | Leveled Compaction | Universal Compaction |
|---------|-------------------|---------------------|
| **读放大** | 低 ✅ | 高 ❌ |
| **写放大** | 高 ❌ | 低 ✅ |
| **空间放大** | 低 ✅ | 高 ❌ |

**选择建议**：
- **读多写少**：Leveled Compaction（默认）
- **写多读少**：Universal Compaction
- **平衡场景**：Leveled Compaction + 优化配置

---

## 二、性能基准

### 2.1 官方 Benchmark（SSD, sync=false）

| 场景 | 读 QPS | 写 QPS | 说明 |
|------|--------|--------|------|
| 高 Cache 命中 + sync=false | 18-25 万 | 18-25 万 | 内存级性能 |
| 读需落盘 + sync=false | 1-5 万 | 18-25 万 | 读受磁盘限制 |
| sync=true（每次 fsync） | — | 7-10 万 | 写受磁盘限制 |
| HDD | 数千-1.5 万 | 1-2 万 | IO 延迟大 |

### 2.2 测试硬件配置参考

| 组件 | 规格 |
|------|------|
| **CPU** | Intel Xeon Gold 6248R @ 2.40GHz, 24 核 (48 线程) |
| **内存** | 256GB DDR4 |
| **硬盘** | NVMe SSD 2TB × 2 (RAID-0, ~5-6 GB/s) |
| **系统** | CentOS 7.9 / Ubuntu 20.04 |

---

## 三、瓶颈原因分析

### 3.1 写入瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **WAL sync** | `sync_log=true` 时每次写都 fsync | 写延迟高、QPS 低 |
| **Immutable MemTable 堆积** | Flush 跟不上，超过 `max_write_buffer_number` | Write Stall |
| **L0 文件过多** | Compaction 跟不上，超过 `level0_slowdown_writes_trigger` | Write Slowdown |
| **Pending Compaction 过大** | 待合并数据量超过 `soft/hard_pending_compaction_bytes_limit` | 写入限流 |
| **单线程 WAL** | WAL 写入串行 | 高并发下成为瓶颈 |

### 3.2 读取瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **L0 文件多** | L0 无序需遍历多个 SST | 点查延迟高 |
| **Block Cache 命中低** | Cache 太小或热点分布变化 | 大量磁盘 IO |
| **无 Bloom Filter** | 无效 SST 也被读入 | 读放大严重 |
| **层级深** | 每层可能查一个 SST，层数多则 IO 多 | 读路径长 |
| **Index/Filter 未缓存** | 每次查 SST 都要读 Index/Filter | 额外 IO |

### 3.3 Compaction 瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **Compaction 线程不足** | `max_background_compactions` 过小 | L0 堆积、Write Stall |
| **写放大高** | Leveled Compaction 导致多次重写 | 磁盘 IO 压力大 |
| **Compaction 与前台争抢 IO** | 无 Rate Limiter | 前台延迟抖动 |
| **单次 Compaction 过大** | 未开启 `max_subcompactions` | CPU 利用率低、耗时长 |

### 3.4 资源瓶颈

| 瓶颈点 | 原因 | 典型表现 |
|--------|------|----------|
| **内存不足** | Block Cache + MemTable + 索引占满 | 频繁换页、性能下降 |
| **磁盘 IOPS 不足** | 写放大 + 读放大叠加 | 延迟抖动、QPS 下降 |
| **CPU 压缩** | 压缩算法过重（如 Zlib） | CPU 成为瓶颈 |
| **磁盘空间** | Compaction 临时空间 + 多版本数据 | 空间不足 |

---

## 四、优化手段

### 4.1 写入优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **WAL sync 策略** | 非强一致场景 `sync_log=false` | 显著降低写延迟 |
| **批量写入** | 使用 WriteBatch 合并多次写入 | 减少 WAL 和锁竞争 |
| **并发 MemTable 写** | 多线程并发写 MemTable | `allow_concurrent_memtable_write=true` |
| **流水线写入** | WAL 与 MemTable 流水线化 | `enable_pipelined_write=true` |
| **增大 MemTable** | 减少 Flush 频率 | `write_buffer_size=256MB` |
| **增加 Flush 线程** | 加快 Immutable 落盘 | `max_background_flushes=2-4` |
| **增加 Compaction 线程** | 控制 L0、避免 Write Stall | `max_background_compactions=4-8` |
| **子 Compaction** | 单次 Compaction 并行化 | `max_subcompactions=4` |
| **Universal Compaction** | 降低写放大 | 写密集型场景 |

### 4.2 读取优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **Block Cache** | 缓存 Data/Index/Filter Block | 建议总内存 1/3 |
| **Index/Filter 缓存** | 避免每次查 SST 都读元数据 | `cache_index_and_filter_blocks=true` |
| **L0 索引常驻** | L0 索引和 Filter 常驻缓存 | `pin_l0_filter_and_index_blocks_in_cache=true` |
| **Bloom Filter** | 快速排除不存在 Key | `NewBloomFilterPolicy(10)` |
| **分区索引** | 7.x+ 降低 Swap 竞争 | `partitioned_index_filters=true` |
| **控制 L0 文件数** | Compaction 及时，减少 L0 |
| **增大 Block Size** | 顺序读友好 | `block_size=16KB-64KB` |

### 4.3 Compaction 优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **动态层级** | 按最深层动态调整各层大小 | `level_compaction_dynamic_level_bytes=true` |
| **优先级策略** | 优先合并重叠小的文件 | `compaction_pri=kMinOverlappingRatio` |
| **Rate Limiter** | 限制 Compaction IO，避免抢前台 | `NewGenericRateLimiter(100MB/s)` |
| **策略选择** | 读多→Leveled，写多→Universal | 按负载选择 |

### 4.4 压缩优化

| 手段 | 说明 | 典型配置 |
|------|------|----------|
| **L0 不压缩** | 减少 CPU | `compression=kNoCompression` |
| **底层 Zstd** | 冷数据高压缩率 | `bottommost_compression=kZstd` |
| **中间层 LZ4** | 平衡速度与压缩率 | `compression=kLZ4` |
| **bytes_per_sync** | 减少同步次数 | `bytes_per_sync=1MB` |

### 4.5 生产环境推荐配置

```cpp
// 基础配置
Options options;
options.max_background_jobs = 6;
options.level_compaction_dynamic_level_bytes = true;
options.bytes_per_sync = 1048576;
options.compaction_pri = kMinOverlappingRatio;

// 表配置
BlockBasedTableOptions table_options;
table_options.block_size = 16 * 1024;  // 16KB
table_options.cache_index_and_filter_blocks = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;
table_options.format_version = 5;
options.table_factory.reset(NewBlockBasedTableFactory(table_options));

// 列族配置
ColumnFamilyOptions cf_options;
cf_options.write_buffer_size = 64 << 20;  // 64MB
cf_options.max_write_buffer_number = 3;
cf_options.compression = kLZ4Compression;
cf_options.bottommost_compression = kZSTDCompression;
```

### 4.6 场景化配置

```cpp
// 写密集型
options.compaction_style = kCompactionStyleUniversal;
cf_options.write_buffer_size = 256 << 20;
cf_options.max_write_buffer_number = 4;

// 读密集型
options.compaction_style = kCompactionStyleLevel;
table_options.block_cache = NewLRUCache(2ULL << 30);  // 2GB
table_options.cache_index_and_filter_blocks = true;

// 内存受限
table_options.partitioned_index_filters = true;
cf_options.write_buffer_size = 16 << 20;
cf_options.max_write_buffer_number = 2;

// 低延迟 / 避免 Write Stall
options.rate_limiter.reset(NewGenericRateLimiter(100 << 20));
```

---

## 五、监控与排查

### 5.1 监控接口

```cpp
// 获取统计信息
std::string value;
db->GetProperty("rocksdb.stats", &value);
db->GetProperty("rocksdb.num-files-at-level0", &value);
db->GetProperty("rocksdb.compaction-pending", &value);
db->GetProperty("rocksdb.is-write-stopped", &value);
db->GetProperty("rocksdb.num-immutable-mem-table", &value);
db->GetProperty("rocksdb.block-cache-hit-rate", &value);
```

### 5.2 关键指标

| 指标 | 关注点 | 告警阈值 |
|------|--------|---------|
| 写延迟 P99 | 是否明显升高 | > 10ms |
| Write Stall 次数/时长 | 是否频繁触发 | > 0 |
| L0 文件数 | 是否持续偏高 | > 4 |
| Pending Compaction Bytes | 是否逼近阈值 | > 256MB |
| Block Cache 命中率 | 是否低于 90% | < 90% |
| 磁盘 IOPS/带宽 | 是否饱和 | > 80% |

### 5.3 写入变慢排查

```
1. is-write-stopped → true? → Write Stall
2. num-immutable-mem-table > 2? → Flush 跟不上
3. num-files-at-level0 > 20? → L0 堆积
4. compaction-pending → true? → Compaction 滞后
5. 调整方向：
   - 增加 Flush/Compaction 线程
   - 调整 Compaction 策略
   - 放宽 sync 配置
```

### 5.4 读取变慢排查

```
1. block-cache-hit-rate < 90%? → 增大 Cache
2. num-files-at-level0 > 4? → Compaction 不及时
3. Bloom Filter 已启用？→ 配置 filter_policy
4. cache_index_and_filter_blocks? → 启用
5. 调整方向：
   - 增大 Block Cache
   - 启用 Index/Filter 缓存
   - 优化 Bloom Filter 配置
```

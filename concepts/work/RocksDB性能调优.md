# RocksDB 性能调优

RocksDB 的性能调优涉及写入、读取、Compaction 等多个维度，需要根据具体场景选择合适的配置。

## 一、写入优化

### 1. 批量写入

使用 WriteBatch 减少频繁 IO：

```cpp
WriteBatch batch;
batch.Put("key1", "value1");
batch.Delete("key2");
db->Write(WriteOptions(), &batch);
```

### 2. 并发配置

- 启用 `allow_concurrent_memtable_write`：在 sync0 模式下性能提升 3 倍，sync1 模式下提升 2 倍
- 启用 `enable_pipelined_write`：形成写入流水线，提高吞吐量

### 3. 参数调优

- 合理设置 `write_buffer_size`（默认 64MB）
- 合理设置 `max_total_wal_size`
- 根据需求配置 `sync_log`：true 确保数据安全（性能损耗），false 依赖操作系统缓存刷盘

## 二、读取优化

### 1. Block Cache

缓存热数据块（LRU/Clock 策略），减少磁盘读取。建议这应该是总内存预算的 1/3 左右。

```cpp
auto cache = NewLRUCache(128 << 20);

BlockBasedTableOptions table_options;
table_options.block_cache = cache;

auto table_factory = new BlockBasedTableFactory(table_options);
cf_options.table_factory.reset(table_factory);
```

### 2. Bloom Filter

快速过滤不存在 Key，减少无效 IO（默认 10 bits/Key）。需配置 `cache_index_and_filter_blocks=true` 缓存 Filter 到内存。

### 3. Table Cache

缓存 SST 元信息（如 Index Block），加速文件定位。

### 4. 分区索引（Partitioned Index/Filter）

7.x 引入的多级索引结构，将索引顶层常驻内存，下层按需加载，减少 Swap 竞争。内存受限场景性能提升可达 10 倍。

## 三、Compaction 优化

### 1. 策略选择

- **Leveled 策略**（默认）：减小空间放大和读放大，适合大多数场景
- **Universal 策略**：减少写放大，适合写密集场景
- **Dynamic Leveled Compaction**：更稳定的空间效率

### 2. 后台线程

增加 `max_background_jobs`（建议设置为 6）提升 Compaction 效率。

### 3. 限制 Compaction I/O 突发

设置 `soft_pending_compaction_bytes_limit` 避免突发 I/O。

## 四、压缩算法

压缩是在 CPU、I/O 和存储空间之间进行权衡：

- `cf_options.compression`：控制第一级的压缩类型，建议使用 LZ4（`kLZ4Compression`），如果不可用则使用 Snappy（`kSnappyCompression`）
- `cf_options.bottommost_compression`：控制最底层的压缩类型，建议使用 ZStandard（`kZSTD`），如果不可用则使用 Zlib（`kZlibCompression`）

## 五、推荐配置

以下设置可为常规工作负载实现合理的开箱即用性能：

```cpp
cf_options.level_compaction_dynamic_level_bytes = true;
opts.max_background_jobs = 6;
options.bytes_per_sync = 1048576;
options.compaction_pri = kMinOverlappingRatio;
table_options.block_size = 16 * 1024;
table_options.cache_index_and_filter_blocks = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;
table_options.format_version = <the latest version>;
```

## 六、场景优化策略

| 场景 | 优化策略 | 配置示例 |
|------|---------|----------|
| 写入瓶颈 | 增大 MemTable 数量、启用压缩、流水线写入 | `setMaxWriteBufferNumber=3`, `enable_pipelined_write=true` |
| 读取延迟高 | 扩大 Block Cache、增加 Bloom Filter 位数、缓存 Index/Filter 块 | `setBlockCache(new LRUCache(256MB))`, `setFilterPolicy(new BloomFilter(12))` |
| Compaction 卡顿 | 增加后台线程、切换 Compaction 策略、动态调整层级大小 | `setIncreaseParallelism(4)`, `setLevelCompactionDynamicLevelBytes=true` |
| 快照备份 | 直接拷贝 SST 文件实现快速扩容 | 使用 rsync/scp 复制 RocksDB 目录 |

## 七、生产建议

- 优先启用 Leveled Compaction 和 Bloom Filter
- 监控 Block Cache 命中率（低于 90% 需扩容）及 Compaction 频率
- 避免频繁启停 DB 实例（长生命周期管理）
- 如果使用闪存，建议使用 discard 标志挂载文件系统，并启用 SST 文件管理器限制文件删除速度

[src: raw/ingested/2技术/rocksdb/rocksdb.md]

## Related Pages
- [[RocksDB写入流程]]
- [[RocksDB读取流程]]
- [[RocksDB Compaction]]
- [[RocksDB Env插件]]
- [[RocksDB版本演进]]

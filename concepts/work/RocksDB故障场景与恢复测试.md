# RocksDB 故障场景与恢复测试

本文档记录 RocksDB 在故障场景下的性能测试数据，包括崩溃恢复时间和 Write Stall 场景测试。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-七、故障场景和恢复测试.md]

## 7.1 崩溃恢复时间测试

### 测试配置
- **故障类型**: 进程崩溃 / 机器重启
- **数据集大小**: 100GB / 500GB / 1TB
- **恢复策略**: 不同配置

### RocksDB 关键配置 (恢复优化配置)

| 配置参数                              | 值          | 说明 |
|---------------------------------------|-------------|------|
| **write_buffer_size**                 | 128MB       | MemTable大小，影响恢复时重放效率 |
| **max_write_buffer_number**           | 4           | 最大MemTable数量 |
| **wal_dir**                           | /tmp/rocksdb_wal | WAL文件目录，影响恢复性能 |
| **wal_ttl_seconds**                   | 0           | WAL生存时间，0表示不自动删除 |
| **wal_size_limit_mb**                 | 0           | WAL大小限制，0表示无限制 |
| **max_background_jobs**               | 8           | 后台线程数，加速恢复过程 |
| **max_total_wal_size**                | 0           | 总WAL大小限制，0表示无限制 |
| **avoid_flush_during_recovery**       | false       | 恢复期间是否避免flush |
| **allow_concurrent_memtable_write**   | true        | 允许并发MemTable写入 |
| **compression_type**                  | snappy      | 压缩算法，影响存储和恢复效率 |
| **compaction_style**                  | Level(0)    | Compaction策略 |
| **level0_file_num_compaction_trigger**| 4           | L0触发compaction阈值 |
| **target_file_size_base**             | 128MB       | SST文件目标大小 |
| **max_bytes_for_level_base**          | 512MB       | L1层最大字节数 |
| **max_background_jobs**               | 8           | 后台线程数，加速恢复过程 |
| **max_background_compactions**        | 2           | 最大compaction并发数 |
| **max_background_flushes**            | 2           | 最大flush并发数 |
| **bytes_per_sync**                    | 1048576     | 同步间隔，影响WAL写入性能 |
| **wal_bytes_per_sync**                | 1048576     | WAL同步间隔 |

### 压测指令
```bash
# 崩溃恢复测试 - 100GB数据集 (先写入数据，然后重启进程测试恢复)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --compression_type=snappy --write_buffer_size=134217728 --wal_dir=/tmp/rocksdb_wal

# 崩溃恢复测试 - 500GB数据集 (先写入数据，然后重启进程测试恢复)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=snappy --write_buffer_size=134217728 --wal_dir=/tmp/rocksdb_wal

# 崩溃恢复测试 - 1TB数据集 (先写入数据，然后重启进程测试恢复)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=100000000 --threads=16 --compression_type=snappy --write_buffer_size=134217728 --wal_dir=/tmp/rocksdb_wal
```

| 数据集大小 | 崩溃恢复时间(秒) | WAL重放时间(秒) | 数据完整性 | CPU峰值使用率 | 内存峰值使用(GB) |
|------------|------------------|-----------------|------------|----------------|------------------|
| 100GB      | 45               | 32              | 100%       | 85%            | 24.5             |
| 500GB      | 185              | 142             | 100%       | 92%            | 68.9             |
| 1TB        | 420              | 365             | 100%       | 95%            | 142.8            |

#### 瓶颈分析

1. **恢复时间随数据量线性增长**：从100GB的45秒增加到1TB的420秒
2. **WAL重放开销巨大**：1TB数据WAL重放需要365秒，是总恢复时间的87%
3. **内存和CPU压力大**：恢复期间CPU使用率高达95%，内存使用142.8GB

**优化建议：**
1. **优化WAL配置**：减小WAL文件大小，启用WAL压缩减少重放时间
2. **增加恢复并发度**：使用更多线程并行重放WAL日志
3. **定期checkpoint**：创建checkpoint减少恢复时需要重放的数据量
4. **使用高速存储**：在NVMe SSD上放置WAL文件加速重放

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-七、故障场景和恢复测试.md]

## 7.2 Write Stall 场景测试

### 测试配置
- **触发条件**: L0 文件数过多
- **监控指标**: 延迟、QPS、队列长度

### RocksDB 关键配置 (Write Stall触发配置)

| 配置参数                              | 值              | 说明 |
|---------------------------------------|-----------------|------|
| **compaction_style**                  | Level(0)        | Compaction策略 |
| **level0_file_num_compaction_trigger** | 8/12/20/50      | L0文件触发compaction的数量阈值 |
| **level0_slowdown_writes_trigger**    | 12/16/28/70     | 触发写减速的L0文件数 |
| **level0_stop_writes_trigger**        | 20/24/40/100    | 触发写停止的L0文件数 |
| **target_file_size_base**             | 64MB            | SST文件目标大小 |
| **max_bytes_for_level_base**          | 256MB           | L1层最大字节数 |
| **write_buffer_size**                 | 64MB            | MemTable大小，影响flush频率 |
| **max_write_buffer_number**           | 3               | 最大MemTable数量 |
| **max_background_jobs**               | 4               | 后台线程数，影响compaction速度 |
| **max_background_compactions**        | 2               | 最大compaction并发数 |
| **target_file_size_base**             | 64MB            | SST文件目标大小 |
| **compression_type**                  | snappy          | 压缩算法，影响compaction性能 |
| **bytes_per_sync**                    | 1048576         | 同步间隔，影响写入性能 |
| **hard_pending_compaction_bytes_limit** | 256MB | 硬限制待compaction字节数 |

### 压测指令
```bash
# Write Stall测试 - L0文件数控制 (调整level0_file_num_compaction_trigger参数)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=snappy --level0_file_num_compaction_trigger=8 --level0_slowdown_writes_trigger=12 --level0_stop_writes_trigger=20

# Write Stall测试 - 轻微Write Stall
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=snappy --level0_file_num_compaction_trigger=12 --level0_slowdown_writes_trigger=16 --level0_stop_writes_trigger=24

# Write Stall测试 - 中等Write Stall
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=snappy --level0_file_num_compaction_trigger=20 --level0_slowdown_writes_trigger=28 --level0_stop_writes_trigger=40

# Write Stall测试 - 严重Write Stall
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=snappy --level0_file_num_compaction_trigger=50 --level0_slowdown_writes_trigger=70 --level0_stop_writes_trigger=100
```

#### 性能测试结果

| L0文件数阈值 | 触发Write Stall | 写QPS下降百分比 | 延迟增加倍数 | 恢复时间(秒) | 用户影响   |
|--------------|-----------------|------------------|--------------|--------------|------------|
| 8            | 否              | 0%               | 1.0x         | -            | 无         |
| 12           | 是，轻微        | 15%              | 2.1x         | 25           | 可接受     |
| 20           | 是，中等        | 35%              | 4.8x         | 85           | 明显卡顿   |
| 50           | 是，严重        | 70%              | 12.5x        | 280          | 服务不可用 |

##### 瓶颈分析

1. **L0文件数累积导致Write Stall**：文件数越多，Stall越严重
2. **恢复时间长**：严重Stall情况下需要280秒恢复，影响可用性
3. **性能下降显著**：严重Stall时写QPS下降70%，延迟增加12.5倍

**优化建议：**
1. **调整L0文件数阈值**：根据硬件性能设置合适的level0_file_num_compaction_trigger
2. **增加compaction线程**：使用更多后台线程加速compaction，减少L0文件累积
3. **升级存储硬件**：使用更高IOPS的SSD减少compaction时间
4. **监控和预警**：实时监控L0文件数，提前触发维护操作

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-七、故障场景和恢复测试.md]

## Related Pages
- [[RocksDB性能分析与瓶颈]]
- [[RocksDB写入流程]]
- [[RocksDB文件体系]]
- [[RocksDB写路径串行化瓶颈]]
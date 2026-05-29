# RocksDB 内存和缓存配置测试

本文档记录 RocksDB 在内存和缓存配置下的性能测试数据，包括 Block Cache 大小影响测试和并发线程数影响测试。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

## 内存和缓存测试硬件环境说明

本节内存和缓存配置测试主要在**基础配置**和**高性能配置**环境下进行，针对不同内存容量和并发处理能力的场景进行测试。

### 基础配置（Block Cache测试环境）
- **CPU**: Intel Xeon Gold 6248R @ 2.40GHz, 24核48线程
- **内存**: 256GB DDR4（支持大Block Cache测试，最大可达64GB）
- **存储**: NVMe SSD 2TB × 2 (RAID-0, 5-6GB/s带宽, ~250k IOPS)
- **网络**: 万兆以太网
- **操作系统**: CentOS 7.9/Ubuntu 20.04
- **适用测试**: Block Cache大小影响测试（1GB-64GB范围）

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 高性能配置（并发线程测试环境）
- **CPU**: AMD EPYC 7742 @ 2.25GHz, 64核128线程（高并发测试）
- **内存**: 512GB DDR4（充足内存支持高并发测试）
- **存储**: NVMe SSD 4TB × 4 (RAID-0, 12-15GB/s带宽, ~500k IOPS)
- **网络**: 25G以太网
- **操作系统**: Ubuntu 22.04
- **适用测试**: 并发线程数影响测试（1-64线程范围）

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 测试环境特点
- **内存容量充足**: 支持大Block Cache和多线程并发测试
- **存储I/O性能优秀**: 确保测试不受存储瓶颈影响，专注内存和缓存效果
- **CPU核数丰富**: 高性能配置64核支持并发线程扩展测试
- **网络性能良好**: 确保测试结果不受网络延迟影响

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 测试策略说明
- **Block Cache测试**: 使用基础配置256GB内存，支持1GB到64GB缓存范围测试
- **并发线程测试**: 使用高性能配置64核CPU，测试1到64线程的并发影响
- **测试数据量**: 100GB-200GB数据集，确保内存配置对性能的影响明显
- **性能指标**: 重点关注QPS、延迟、CPU使用率、缓存命中率等内存相关指标

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

---

## 4.1 Block Cache 大小影响测试

### 测试配置
- **工作负载**: 80% 随机读 + 20% 随机写
- **数据集**: 200GB
- **Block Cache**: 1GB / 4GB / 16GB / 64GB

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### RocksDB 关键配置 (Block Cache外固定配置)

| 配置参数                              | 值     | 说明 |
|---------------------------------------|--------|------|
| **write_buffer_size**                 | 64MB   | MemTable大小，测试中保持固定 |
| **max_write_buffer_number**           | 3      | 最大MemTable数量，保持固定 |
| **compression_type**                  | snappy | 压缩算法，影响数据大小和读取性能 |
| **block_size**                        | 4KB    | 数据块大小，影响缓存粒度和I/O效率 |
| **cache_index_and_filter_blocks**     | true   | 缓存索引和过滤器块，优化查找性能 |
| **compaction_style**                  | Level(0)    | Compaction策略 |
| **level0_file_num_compaction_trigger**| 4           | L0触发compaction阈值 |
| **max_background_jobs**               | 4           | 后台线程数，保持并发处理能力 |
| **max_background_compactions**        | 2           | 最大compaction并发数 |
| **max_background_flushes**            | 1           | 最大flush并发数 |
| **max_open_files**                    | -1          | 最大打开文件数，无限制 |
| **table_cache_numshardbits**         | 4           | Table Cache分片数，影响并发访问 |
| **read_cache_size**                   | 0           | 读取缓存大小，禁用以专注Block Cache测试 |

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 压测指令
```bash
# Block Cache 测试 - 1GB (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=20 --block_cache=1073741824 --compression_type=snappy

# Block Cache 测试 - 4GB (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=20 --block_cache=4294967296 --compression_type=snappy

# Block Cache 测试 - 16GB (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=20 --block_cache=17179869184 --compression_type=snappy

# Block Cache 测试 - 64GB (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=20 --block_cache=68719476736 --compression_type=snappy
```

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 性能测试结果

| Block Cache大小 | 读QPS    | 读P95延迟(ms) | 缓存命中率 | 内存使用(GB) | 读放大 | IOPS减少百分比 |
|-----------------|----------|----------------|------------|--------------|--------|----------------|
| 1GB             | 45,680   | 4.2             | 65.8%      | 2.1          | 8.5x   | 0% (基准)      |
| 4GB             | 68,920   | 2.8             | 82.4%      | 5.2          | 5.2x   | 34%            |
| 16GB            | 89,450   | 1.9             | 91.6%      | 18.4         | 3.8x   | 49%            |
| 64GB            | 95,680   | 1.6             | 94.2%      | 66.8         | 3.2x   | 52%            |

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

#### 瓶颈分析

1. **小缓存命中率低**（1GB）：导致大量磁盘I/O和8.5x读放大
2. **大缓存收益递减**（64GB）：从16GB到64GB只提升3%的IOPS
3. **内存使用与性能平衡**：64GB缓存消耗大量内存但性能提升有限

**优化建议：**
1. **根据数据集大小配置**：缓存大小应为数据集的1-2倍以达到最佳效果
2. **监控缓存效果**：根据实际命中率调整缓存大小，避免浪费内存
3. **考虑分层缓存**：结合操作系统页缓存和Block Cache优化
4. **使用压缩缓存**：考虑缓存压缩减少内存占用同时保持性能

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

## 4.2 并发线程数影响测试

### 测试配置
- **工作负载**: 50% 读 + 50% 写
- **并发线程数**: 1 / 4 / 16 / 64
- **数据集**: 100GB

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### RocksDB 关键配置 (并发线程数外固定配置)

| 配置参数                              | 值     | 说明 |
|---------------------------------------|--------|------|
| **block_cache**                       | 8GB    | Block Cache大小，测试中保持固定 |
| **write_buffer_size**                 | 64MB   | MemTable大小，影响写性能 |
| **max_write_buffer_number**           | 3      | 最大MemTable数量，保持固定 |
| **compression_type**                  | snappy | 压缩算法，影响CPU使用率 |
| **max_background_jobs**               | 8      | 后台线程数，处理compaction和flush |
| **max_background_compactions**        | 2      | 最大compaction并发数 |
| **max_background_flushes**            | 1      | 最大flush并发数 |
| **compaction_style**                  | Level(0)    | Compaction策略 |
| **level0_file_num_compaction_trigger** | 4      | L0文件触发compaction阈值 |
| **level0_slowdown_writes_trigger**    | 20          | 触发写减速的L0文件数 |
| **level0_stop_writes_trigger**        | 36          | 触发写停止的L0文件数 |
| **target_file_size_base**             | 64MB        | SST文件目标大小 |
| **max_bytes_for_level_base**          | 256MB       | L1层最大字节数 |
| **table_cache_numshardbits**          | 4           | Table Cache分片数，影响并发访问效率 |
| **cache_index_and_filter_blocks**     | true        | 缓存索引和过滤器块 |

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 压测指令
```bash
# 并发线程数测试 - 1线程 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=10000000 --threads=1 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 并发线程数测试 - 4线程 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=10000000 --threads=4 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 并发线程数测试 - 16线程 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=10000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 并发线程数测试 - 64线程 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=10000000 --threads=64 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592
```

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

### 性能测试结果

| 并发线程数 | 总QPS    | 读QPS   | 写QPS   | 读P95延迟(ms) | 写P95延迟(ms) | CPU使用率 | 锁竞争百分比 |
|------------|----------|---------|---------|----------------|----------------|-----------|--------------|
| 1          | 25,680   | 12,840  | 12,840  | 3.2             | 8.5             | 25%       | 0%           |
| 4          | 89,450   | 44,725  | 44,725  | 2.1             | 5.8             | 68%       | 5%           |
| 16         | 142,380  | 71,190  | 71,190  | 1.8             | 4.9             | 85%       | 12%          |
| 64         | 165,920  | 82,960  | 82,960  | 2.2             | 5.2             | 92%       | 28%          |

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

#### 瓶颈分析

1. **单线程性能低**：1线程时CPU利用率仅25%，说明单线程无法发挥硬件性能
2. **高并发锁竞争增加**：64线程时锁竞争达到28%，导致性能提升受限
3. **线程数与CPU核数匹配**：16线程时性能较好，64线程时出现竞争

**优化建议：**
1. **根据CPU核数配置线程**：线程数应略高于CPU核数，避免过度竞争
2. **优化锁机制**：使用更细粒度的锁或无锁数据结构减少竞争
3. **考虑线程亲和性**：将线程绑定到特定CPU核心提升缓存局部性
4. **监控锁竞争**：通过perf工具分析锁热点并优化

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-四、内存和缓存配置测试.md]

## Related Pages
- [[RocksDB配置参数性能测试]]
- [[RocksDB存储介质对比测试]]
- [[RocksDB性能分析与瓶颈]]
- [[RocksDB文件体系]]
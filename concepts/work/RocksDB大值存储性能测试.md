# RocksDB 大值存储性能测试

本文档记录 RocksDB 在大值存储场景下的性能测试数据，包括 1KB、10KB、100KB、1MB 值大小的测试结果和瓶颈分析。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-六、特殊场景性能测试.md]

## 测试配置
- **值大小**: 1KB / 10KB / 100KB / 1MB
- **工作负载**: 随机读写
- **数据集**: 100GB 总大小

## RocksDB 关键配置 (大值存储优化)

| 配置参数                              | 值      | 说明 |
|---------------------------------------|---------|------|
| **block_cache**                       | 8GB     | Block Cache大小，考虑大值对缓存的影响 |
| **write_buffer_size**                 | 256MB   | 增大MemTable适应大值写入 |
| **max_write_buffer_number**           | 4       | 增加MemTable数量 |
| **compression_type**                  | snappy  | 压缩算法，大值通常压缩效果更好 |
| **block_size**                        | 64KB    | 增大block size适应大值存储 |
| **max_background_jobs**               | 6       | 后台线程数，处理大值compaction |
| **target_file_size_base**             | 512MB   | 增大SST文件大小，减少大值文件的数量 |
| **cache_index_and_filter_blocks**     | true    | 缓存索引和过滤器块 |
| **bloom_bits**                        | 10      | Bloom过滤器位数，帮助大值查找 |
| **readahead_size**                    | 16384   | 预读大小，对大值读取优化 |
| **table_cache_numshardbits**          | 4       | Table Cache分片数 |
| **compaction_style**                  | Level(0)| Compaction策略 |
| **level0_file_num_compaction_trigger**| 4       | L0触发compaction阈值 |
| **max_bytes_for_level_base**          | 2GB     | L1层最大字节数 |
| **max_background_compactions**        | 2       | 最大compaction并发数 |
| **max_background_flushes**            | 1       | 最大flush并发数 |

## 压测指令
```bash
# 大值存储测试 - 1KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=1024 --num=100000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 大值存储测试 - 10KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=10240 --num=10000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 大值存储测试 - 100KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=102400 --num=1000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 大值存储测试 - 1MB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=1048576 --num=100000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592
```

## 性能测试结果

| 值大小 | 读QPS   | 写QPS   | 读P95延迟(ms) | 写P95延迟(ms) | 存储效率 | 网络开销占比 |
|--------|---------|---------|----------------|----------------|----------|--------------|
| 1KB    | 95,680  | 68,920  | 1.8             | 4.2             | 85%      | 15%          |
| 10KB   | 45,680  | 32,450  | 3.5             | 8.8             | 92%      | 45%          |
| 100KB  | 12,450  | 8,920   | 8.2             | 18.5            | 95%      | 78%          |
| 1MB    | 2,850   | 1,680   | 25.8            | 45.2            | 96%      | 92%          |

### 瓶颈分析

1. **大值存储性能急剧下降**：从1KB到1MB，QPS下降96%以上
2. **网络开销成为主要瓶颈**：1MB值网络开销占比92%
3. **存储效率提升但性能牺牲**：大值压缩率高但读取性能差

**优化建议：**
1. **避免存储过大值**：将大值分离存储，使用引用或外部存储
2. **优化网络配置**：使用更高带宽网络或压缩传输减少网络开销
3. **调整Block Cache策略**：对大值禁用缓存或使用专门的缓存策略
4. **考虑值分离**：使用BlobDB或类似机制分离大值存储

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-六、特殊场景性能测试.md]
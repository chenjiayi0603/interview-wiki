# RocksDB 长尾延迟优化测试

本文档记录 RocksDB 在长尾延迟优化场景下的测试数据，包括不同优化策略（更大Block Cache、并发优化、存储优化等）对P99/P99.9延迟的影响。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-六、特殊场景性能测试.md]

## 测试配置
- **优化策略**: 不同参数组合
- **工作负载**: 混合读写
- **关注指标**: P99, P99.9 延迟

## RocksDB 关键配置 (逐步优化配置)

| 配置参数                              | 默认值  | 优化值           | 说明 |
|---------------------------------------|---------|------------------|------|
| **block_cache**                       | 8GB     | 16GB→32GB→64GB   | 逐步增加缓存减少I/O抖动 |
| **max_background_jobs**               | 4       | 8→12             | 增加后台线程减少compaction干扰 |
| **bytes_per_sync**                    | 1MB     | 1MB→4MB          | 增大同步间隔减少同步开销 |
| **write_buffer_size**                 | 64MB    | 128MB            | 增大MemTable减少flush频率 |
| **compaction_style**                  | Level(0) | Level(0)         | Compaction策略 |
| **level0_file_num_compaction_trigger** | 4      | 8                | 提高L0触发阈值减少compaction |
| **level0_slowdown_writes_trigger**    | 20      | 32               | 触发写减速的L0文件数 |
| **level0_stop_writes_trigger**        | 36      | 64               | 触发写停止的L0文件数 |
| **max_background_compactions**        | 2       | 4                | 增加compaction并发数 |
| **max_background_flushes**            | 1       | 2                | 最大flush并发数 |
| **compression_type**                  | snappy  | snappy           | 保持压缩算法一致 |
| **cache_index_and_filter_blocks**    | true    | true             | 缓存索引和过滤器块 |
| **table_cache_numshardbits**          | 4       | 6                | 增加Table Cache分片 |
| **read_cache_size**                   | 0       | 0                | 禁用读取缓存专注Block Cache |

## 压测指令
```bash
# 长尾延迟优化测试 - 默认配置 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=100000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 长尾延迟优化测试 - +更大Block Cache (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=100000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=17179869184

# 长尾延迟优化测试 - +并发优化 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=100000000 --threads=32 --readwritepercent=50 --compression_type=snappy --block_cache=17179869184 --max_background_jobs=8

# 长尾延迟优化测试 - +存储优化 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=100000000 --threads=32 --readwritepercent=50 --compression_type=snappy --block_cache=17179869184 --max_background_jobs=8 --bytes_per_sync=1048576

# 长尾延迟优化测试 - 全部优化 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=100000000 --threads=32 --readwritepercent=50 --compression_type=snappy --block_cache=34359738368 --max_background_jobs=12 --bytes_per_sync=1048576 --write_buffer_size=268435456
```

## 性能测试结果

| 优化策略         | P50延迟(ms) | P95延迟(ms) | P99延迟(ms) | P99.9延迟(ms) | 平均QPS  | CPU使用率 |
|------------------|-------------|-------------|-------------|---------------|----------|-----------|
| 默认配置         | 1.2         | 4.5         | 12.8        | 45.6          | 125,680  | 72%       |
| +更大Block Cache | 1.1         | 3.8         | 10.2        | 35.8          | 132,450  | 68%       |
| +并发优化        | 1.0         | 3.2         | 8.9         | 28.4          | 145,920  | 75%       |
| +存储优化        | 0.9         | 2.8         | 7.5         | 22.1          | 152,680  | 78%       |
| 全部优化         | 0.8         | 2.2         | 6.2         | 18.9          | 165,420  | 82%       |

### 瓶颈分析

1. **默认配置长尾延迟严重**：P99.9延迟达45.6ms，影响用户体验
2. **缓存不足导致抖动**：Block Cache小导致频繁磁盘I/O引起延迟尖峰
3. **后台任务干扰**：compaction和flush操作影响前台请求延迟

**优化建议：**
1. **增加Block Cache**：使用更大缓存减少磁盘I/O抖动
2. **调整后台任务优先级**：降低compaction线程优先级减少对前台请求干扰
3. **启用多队列**：分离读写队列减少相互阻塞
4. **使用更快的存储**：NVMe SSD减少I/O延迟波动

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-六、特殊场景性能测试.md]
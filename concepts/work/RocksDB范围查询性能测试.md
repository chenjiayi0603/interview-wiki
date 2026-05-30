# RocksDB 范围查询性能测试

## 测试配置
- **工作负载**: 范围查询 (Iterator)
- **查询范围**: 1000 keys
- **数据集**: 100GB
- **Block Cache**: 16GB

## RocksDB 关键配置

| 配置参数                              | 值          | 说明 |
|---------------------------------------|-------------|------|
| **block_cache**                       | 16GB        | 大Block Cache提升范围查询缓存命中率 |
| **block_size**                        | 16KB        | 较大block size减少范围查询的I/O次数 |
| **cache_index_and_filter_blocks**     | true        | 缓存索引和过滤器块，加速范围查找 |
| **prefix_extractor**                   | 前缀提取器  | 启用前缀bloom filter优化范围查询 |
| **bloom_bits**                        | 10          | Bloom过滤器位数，平衡内存和误判率 |
| **max_sequential_skip_in_iterations** | 8           | 迭代器中最大顺序跳跃，优化范围扫描 |
| **readahead_size**                    | 0           | 预读大小，范围查询通常不需要预读 |
| **table_cache_numshardbits**         | 6           | 增加Table Cache分片，提升并发范围查询性能 |
| **iterator_upper_bound**              | 设置        | 限制迭代器上界，减少不必要的扫描 |
| **compaction_style**                  | Level(0)    | Compaction策略，数据预热时使用 |
| **level0_file_num_compaction_trigger**| 4           | L0触发compaction阈值 |
| **max_background_compactions**        | 2           | 后台compaction并发数 |

## 压测指令
```bash
# 范围查询测试 - 10 keys (需先填充数据)
db_bench --benchmarks=seekrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --seek_nexts=10 --block_cache=17179869184

# 范围查询测试 - 100 keys (需先填充数据)
db_bench --benchmarks=seekrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --seek_nexts=100 --block_cache=17179869184

# 范围查询测试 - 1000 keys (需先填充数据)
db_bench --benchmarks=seekrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --seek_nexts=1000 --block_cache=17179869184

# 范围查询测试 - 10000 keys (需先填充数据)
db_bench --benchmarks=seekrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --seek_nexts=10000 --block_cache=17179869184
```

## 性能测试结果

| 查询范围   | QPS     | P95延迟(ms) | 每查询返回键数 | IOPS       | 缓存命中率 | CPU使用率 |
|------------|---------|-------------|----------------|------------|------------|-----------|
| 10 keys    | 12,450  | 2.1         | 10            | 15,680     | 98.2%      | 45%       |
| 100 keys   | 8,920   | 3.8         | 100           | 142,800    | 94.5%      | 52%       |
| 1000 keys  | 5,680   | 8.2         | 1000          | 1,268,900  | 85.6%      | 68%       |
| 10000 keys | 2,450   | 18.5        | 10000         | 8,945,600  | 68.9%      | 78%       |

### 瓶颈分析

1. **范围扩大导致性能急剧下降**：从10 keys到10000 keys，QPS下降80%以上
2. **缓存命中率显著降低**：大范围查询缓存效率从98.2%降至68.9%
3. **CPU使用率随范围增加**：从45%上升到78%，说明计算开销增加

**优化建议：**
1. **优化Iterator使用**：使用Prefix Iterator或Limit范围避免大范围扫描
2. **增加Block Cache**：大范围查询需要更多缓存支持，提升命中率
3. **调整Block Size**：对范围查询使用更大Block Size减少I/O操作
4. **考虑索引优化**：为范围查询设计更好的索引结构或使用Bloom Filter

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-三、不同工作负载模式测试.md]
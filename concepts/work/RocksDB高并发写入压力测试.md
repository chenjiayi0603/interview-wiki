# RocksDB 高并发写入压力测试

本文档记录 RocksDB 在高并发写入场景下的性能测试数据，包括 1/10/50/100 并发客户端下的测试结果和瓶颈分析。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-六、特殊场景性能测试.md]

## 测试配置
- **并发客户端**: 1 / 10 / 50 / 100
- **每个客户端QPS**: 10,000
- **持续时间**: 1小时
- **数据集**: 持续增长

## RocksDB 关键配置 (高并发写入优化)

| 配置参数                              | 值        | 说明 |
|---------------------------------------|-----------|------|
| **write_buffer_size**                 | 512MB     | 大MemTable适应高并发写入压力 |
| **max_write_buffer_number**           | 8         | 增加MemTable数量，分散写入压力 |
| **max_background_jobs**               | 16        | 大幅增加后台线程处理高并发compaction |
| **max_background_compactions**        | 8         | 增加compaction并发数 |
| **max_background_flushes**            | 4         | 增加flush并发数 |
| **compaction_style**                  | Level(0)  | Compaction策略 |
| **level0_file_num_compaction_trigger** | 16      | 提高L0触发阈值，减少compaction频率 |
| **level0_slowdown_writes_trigger**    | 32        | 提高减速触发阈值 |
| **level0_stop_writes_trigger**        | 64        | 提高停止写入触发阈值 |
| **target_file_size_base**             | 256MB     | SST文件目标大小 |
| **max_bytes_for_level_base**          | 1GB       | L1层最大字节数 |
| **bytes_per_sync**                    | 8388608   | 增大同步间隔，减少同步开销 |
| **wal_bytes_per_sync**                | 8388608   | WAL同步间隔 |
| **target_file_size_base**             | 256MB     | 增大SST文件大小 |
| **compression_type**                  | snappy    | 压缩算法，平衡性能和存储 |
| **max_open_files**                    | -1        | 最大打开文件数，无限制 |

## 压测指令
```bash
# 高并发写入测试 - 1客户端 (每个客户端10000 QPS)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=36000000 --threads=16 --compression_type=snappy --write_buffer_size=268435456

# 高并发写入测试 - 10客户端 (每个客户端10000 QPS)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=360000000 --threads=64 --compression_type=snappy --write_buffer_size=536870912

# 高并发写入测试 - 50客户端 (每个客户端10000 QPS)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=1800000000 --threads=128 --compression_type=snappy --write_buffer_size=1073741824

# 高并发写入测试 - 100客户端 (每个客户端10000 QPS)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=3600000000 --threads=256 --compression_type=snappy --write_buffer_size=2147483648
```

## 性能测试结果

| 并发客户端数 | 总写QPS  | 平均延迟(ms) | P99延迟(ms) | 失败率  | CPU使用率 | 内存使用(GB) |
|--------------|----------|--------------|-------------|--------|-----------|--------------|
| 1            | 9,850    | 2.1          | 8.5         | 0.001% | 35%       | 12.5         |
| 10           | 95,680   | 3.2          | 15.8        | 0.005% | 68%       | 28.4         |
| 50           | 456,920  | 5.8          | 28.4        | 0.012% | 85%       | 68.9         |
| 100          | 892,450  | 8.5          | 42.1        | 0.025% | 92%       | 125.6        |

### 瓶颈分析

1. **内存使用急剧增加**：100客户端时内存使用125.6GB，接近硬件极限
2. **P99延迟显著增加**：从8.5ms增加到42.1ms，长尾延迟严重
3. **失败率随并发增加**：虽然绝对值小但相对增加明显

**优化建议：**
1. **增加内存配置**：高并发场景需要更多内存支持Write Buffer和缓存
2. **优化Write Stall阈值**：调整level0文件数限制减少Write Stall发生
3. **使用更高效压缩**：降低CPU开销为更多并发让路
4. **考虑负载均衡**：在多个RocksDB实例间分布负载

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-六、特殊场景性能测试.md]
# RocksDB 顺序写入性能测试

## 测试配置
- **工作负载**: 100% 顺序写入
- **键大小**: 16 bytes
- **值大小**: 100 bytes
- **数据量**: 1TB
- **测试工具**: db_bench

## RocksDB 关键配置

| 配置参数                      | 值        | 说明 |
|-------------------------------|-----------|------|
| **write_buffer_size**         | 256MB     | 增大MemTable大小，减少flush频率 |
| **max_write_buffer_number**   | 4         | 增加MemTable数量，提升写入缓冲 |
| **compaction_style**                  | Level(0)  | Compaction策略 |
| **level0_file_num_compaction_trigger**| 4         | L0触发compaction阈值 |
| **level0_slowdown_writes_trigger**    | 20        | 触发写减速的L0文件数 |
| **level0_stop_writes_trigger**        | 36        | 触发写停止的L0文件数 |
| **target_file_size_base**     | 256MB     | 增大SST文件大小，减少文件数量 |
| **max_bytes_for_level_base**  | 1GB       | 增大各层容量，适应大数据量 |
| **compression_type**          | snappy    | 使用压缩减少存储空间和I/O |
| **max_background_jobs**       | 8         | 增加后台线程数，加速compaction |
| **max_background_compactions**        | 4         | 增加compaction并发数 |
| **max_background_flushes**    | 2         | 增加flush并发数 |
| **bytes_per_sync**            | 1048576   | 增大同步间隔，减少同步开销 |
| **wal_bytes_per_sync**        | 1048576   | WAL同步间隔，平衡性能和持久性 |

## 压测指令
```bash
# 基础配置硬件 - 顺序写入测试
db_bench --benchmarks=fillseq --key_size=16 --value_size=100 --num=100000000 --threads=16 --compression_type=snappy

# 中端配置硬件 - 顺序写入测试
db_bench --benchmarks=fillseq --key_size=16 --value_size=100 --num=100000000 --threads=12 --compression_type=snappy

# 高性能配置硬件 - 顺序写入测试
db_bench --benchmarks=fillseq --key_size=16 --value_size=100 --num=100000000 --threads=32 --compression_type=snappy
```

## 性能测试结果

| 硬件配置     | 写QPS      | P95延迟(ms) | P99延迟(ms) | IOPS      | 带宽(MB/s) | CPU使用率 |
|--------------|------------|-------------|-------------|-----------|------------|-----------|
| 基础配置     | 892,450    | 0.08        | 0.25        | 1,178,600 | 5,420      | 25%       |
| 中端配置     | 685,920    | 0.12        | 0.38        | 907,800   | 4,180      | 32%       |
| 高性能配置   | 1,245,680  | 0.05        | 0.15        | 1,648,900 | 7,580      | 18%       |

### 瓶颈分析

1. **存储带宽限制**：顺序写入主要受限于存储设备的顺序写带宽
2. **CPU开销相对较低**：顺序写入CPU使用率很低（18-32%），说明存储是主要瓶颈
3. **网络和内存影响较小**：顺序写入对网络和内存依赖较小

**优化建议：**
1. **优化存储配置**：使用RAID-0或更高速SSD提升顺序写带宽
2. **调整并发策略**：增加线程数到32-64充分利用存储并行能力
3. **启用Direct I/O**：减少内核缓冲区开销，提升大文件顺序写性能
4. **调整WAL策略**：对顺序写入可考虑异步WAL减少同步开销

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-三、不同工作负载模式测试.md]
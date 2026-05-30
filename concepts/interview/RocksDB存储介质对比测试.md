# RocksDB 存储介质对比测试

本文档记录 RocksDB 在不同存储介质（SSD 类型和云存储）下的性能测试数据，包括 SATA SSD、NVMe Gen3/Gen4、Optane SSD 以及 AWS/Azure/GCP/华为云等云存储的对比测试。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-五、存储介质对比测试.md]

## 5.1 SSD 类型对比测试

### 测试配置
- **工作负载**: 70% 随机读 + 30% 随机写
- **数据集**: 500GB
- **RocksDB版本**: 8.5.3

#### RocksDB 关键配置 (所有SSD类型保持一致)

| 配置参数                              | 值      | 说明 |
|---------------------------------------|---------|------|
| **block_cache**                       | 16GB    | Block Cache大小，充足缓存以突出存储性能差异 |
| **write_buffer_size**                 | 128MB   | MemTable大小，平衡内存使用 |
| **max_write_buffer_number**           | 4       | 最大MemTable数量 |
| **compression_type**                  | snappy  | 压缩算法，所有测试保持一致 |
| **compaction_style**                  | Level(0)| Compaction策略 |
| **level0_file_num_compaction_trigger** | 4       | L0文件触发compaction阈值 |
| **level0_slowdown_writes_trigger**    | 20      | 触发写减速的L0文件数 |
| **level0_stop_writes_trigger**        | 36      | 触发写停止的L0文件数 |
| **target_file_size_base**             | 128MB   | SST文件目标大小 |
| **max_bytes_for_level_base**          | 512MB   | 各层最大字节数 |
| **max_background_jobs**               | 8       | 后台线程数，充分利用存储并发能力 |
| **max_background_compactions**        | 4       | 最大compaction并发数 |
| **max_background_flushes**            | 2       | 最大flush并发数 |
| **cache_index_and_filter_blocks**     | true    | 缓存索引和过滤器块 |
| **table_cache_numshardbits**          | 6       | Table Cache分片数，增加并发性能 |

#### 压测指令
```bash
# SSD类型测试 - SATA SSD (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=50000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_cache=8589934592

# SSD类型测试 - NVMe Gen3 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=50000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_cache=8589934592

# SSD类型测试 - NVMe Gen4 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=50000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_cache=8589934592

# SSD类型测试 - Optane SSD (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=50000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_cache=8589934592
```

#### 性能测试结果

| SSD类型     | 顺序读带宽(MB/s) | 随机读IOPS | 写QPS    | 读QPS    | 读P95延迟(ms) | 写P95延迟(ms) | 价格($/GB) |
|-------------|------------------|------------|----------|----------|----------------|----------------|------------|
| SATA SSD    | 550              | 85,000     | 45,680   | 28,920   | 4.2             | 8.5             | 0.08       |
| NVMe Gen3   | 3,200            | 280,000    | 142,380  | 98,650   | 1.8             | 4.2             | 0.15       |
| NVMe Gen4   | 7,000            | 650,000    | 245,680  | 165,920  | 1.2             | 2.8             | 0.25       |
| Optane SSD  | 2,400            | 550,000    | 189,450  | 142,380  | 0.8             | 2.1             | 1.20       |

##### 瓶颈分析

1. **SATA SSD性能全面落后**：顺序读带宽和随机IOPS限制整体性能
2. **NVMe Gen4性能最优**：高带宽和高IOPS带来最佳性能，但成本较高
3. **Optane SSD延迟最低**：虽然带宽不如NVMe Gen4，但延迟优势明显，适合延迟敏感应用

**优化建议：**
1. **根据预算和需求选择**：性能优先选NVMe Gen4，成本敏感选NVMe Gen3，低延迟选Optane
2. **考虑RAID配置**：使用多盘RAID-0提升总IOPS和带宽
3. **优化队列深度**：根据SSD特性调整I/O队列深度以充分利用性能
4. **监控SSD寿命**：定期检查SSD健康状态，及时更换老化设备

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-五、存储介质对比测试.md]

## 5.2 云存储性能测试

### 测试配置
- **云厂商**: AWS / Azure / GCP
- **实例类型**: 优化存储实例
- **数据集**: 200GB

#### RocksDB 关键配置 (云存储优化配置)

| 配置参数                              | 值        | 说明 |
|---------------------------------------|-----------|------|
| **block_cache**                       | 16GB      | 大Block Cache弥补云存储延迟 |
| **write_buffer_size**                 | 256MB     | 增大MemTable减少云存储写入频率 |
| **max_write_buffer_number**           | 6         | 增加MemTable数量，提升缓冲能力 |
| **compression_type**                  | snappy    | 压缩减少网络传输数据量 |
| **max_background_jobs**               | 6         | 后台线程数，适应云环境并发限制 |
| **max_background_compactions**        | 2         | 控制compaction并发，避免IOPS过载 |
| **bytes_per_sync**                    | 4194304   | 增大同步间隔，减少云存储同步开销 |
| **wal_bytes_per_sync**                | 4194304   | WAL同步间隔，平衡性能和持久性 |
| **compaction_style**                  | Level(0)    | Compaction策略 |
| **level0_file_num_compaction_trigger** | 8       | 提高L0触发阈值，减少compaction频率 |
| **level0_slowdown_writes_trigger**    | 32          | 触发写减速的L0文件数 |
| **level0_stop_writes_trigger**        | 64          | 触发写停止的L0文件数 |
| **target_file_size_base**             | 256MB       | 增大SST文件大小，减少文件操作 |
| **max_bytes_for_level_base**          | 1GB         | L1层最大字节数 |
| **max_background_flushes**            | 2           | 最大flush并发数 |
| **max_open_files**                    | 1000        | 限制打开文件数，适应云环境限制 |
| **table_cache_numshardbits**          | 4           | Table Cache分片数 |

#### 压测指令
```bash
# 云存储测试 - AWS EBS gp3 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 云存储测试 - AWS EBS io2 (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 云存储测试 - Azure Premium (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 云存储测试 - GCP SSD (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 云存储测试 - 华为云 EVS 超高IO (需先填充数据)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592

# 云存储测试 - 华为云 OBS (需先填充数据，适合冷数据/归档场景)
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=50 --compression_type=snappy --block_cache=8589934592
```

#### 性能测试结果

| 云服务       | 存储类型 | 读QPS   | 写QPS   | 读P95延迟(ms) | 写P95延迟(ms) | 成本($/月) | IOPS限制 |
|--------------|----------|---------|---------|----------------|----------------|------------|----------|
| AWS EBS      | gp3      | 45,680  | 28,920  | 3.8             | 12.5            | 120        | 16,000   |
| AWS EBS      | io2      | 68,920  | 42,650  | 2.9             | 8.4             | 580        | 64,000   |
| Azure Premium| P30      | 52,450  | 32,180  | 3.2             | 10.8            | 135        | 5,000    |
| GCP SSD      | pd-ssd   | 58,920  | 38,450  | 2.8             | 9.2             | 110        | 15,000   |
| 华为云 EVS   | 超高IO   | 72,850  | 45,680  | 2.2             | 6.8             | 95         | 50,000   |
| 华为云 OBS   | 标准存储 | 8,420   | 5,280   | 45.2            | 68.5            | 25         | 无限制   |

华为云 EVS 是远端块存储，把卷挂到 ECS 后，RocksDB 像写本地盘一样写 /data/rocksdb；虽然走网络，但 EVS 提供低延迟随机读写，适合在线业务。  
华为云 OBS 是对象存储，给数据库当"冷数据仓库"用：旧 SST 文件、备份、日志定时上传，按 GB 计费，不保证低延迟，适合归档和大数据分析。  

##### 瓶颈分析

1. **网络延迟影响性能**：云存储的网络往返时间导致延迟高于本地SSD
2. **IOPS限制严格**：AWS gp3只有16,000 IOPS，Azure P30仅5,000 IOPS，华为云EVS超高IO提供50k IOPS优势显著
3. **成本与性能权衡**：高性能云盘（如AWS io2）成本是基础云盘的4-5倍，华为云EVS性价比更优
4. **华为云EVS优势**：EVS超高IO在50k IOPS下读写QPS显著优于AWS gp3，延迟更低，成本更低
5. **华为云OBS特点**：对象存储OBS延迟高（45-68ms），不适合随机小I/O，但成本极低（$25/月），吞吐量无限制，适合冷数据/归档/备份场景

**优化建议：**
1. **选择合适云盘类型**：根据IOPS需求选择云盘规格，华为云EVS超高IO适合高性能场景，AWS/Azure/GCP根据预算选择
2. **华为云分层存储策略**：热数据用EVS超高IO，温数据用EVS普通，冷数据用OBS，平衡性能和成本
3. **OBS对象存储应用**：适合大文件顺序读写、数据归档、备份恢复场景，不适合OLTP随机小I/O
4. **使用本地缓存**：在云服务器上配置本地NVMe或DCS作为缓存层，OBS存储冷数据
5. **优化网络配置**：使用华为云内网或专线减少EVS/OBS访问延迟
6. **考虑多盘分布式**：使用多个云盘并行I/O分散负载，OBS适合大数据量顺序扫描

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-五、存储介质对比测试.md]

## Related Pages
- [[RocksDB配置参数性能测试]]
- [[RocksDB性能分析与瓶颈]]
- [[OBS对接RocksDB性能分析]]
- [[RocksDB文件体系]]
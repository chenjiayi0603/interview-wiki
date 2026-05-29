# RocksDB 配置参数性能测试

本文档记录 RocksDB 在不同配置参数下的性能测试数据，包括 Block Size、Write Buffer Size、压缩算法和 Compaction 策略的对比测试。

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-二、不同配置参数的影响测试.md]

## 测试硬件环境

所有配置参数测试均在**基础配置**硬件环境下进行。

### 基础配置详细规格
- **CPU**: Intel Xeon Gold 6248R @ 2.40GHz, 24核48线程
- **内存**: 256GB DDR4
- **存储**: NVMe SSD 2TB × 2 (RAID-0配置)
- **存储性能**: 5-6GB/s 顺序读写带宽, ~250k IOPS 随机读写
- **网络**: 万兆以太网
- **操作系统**: CentOS 7.9/Ubuntu 20.04
- **RocksDB版本**: 8.5.3 (默认，除非特殊说明)

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-二、不同配置参数的影响测试.md]

---

## 2.1 Block Size 参数测试

### 测试配置
- **工作负载**: 70% 读 + 30% 写
- **Block Size**: 4KB / 16KB / 64KB / 256KB
- **数据集**: 200GB
- **硬件**: 基础配置

### 压测指令
```bash
# Block Size 测试 - 4KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_size=4096 --block_cache=8589934592

# Block Size 测试 - 16KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_size=16384 --block_cache=8589934592

# Block Size 测试 - 64KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_size=65536 --block_cache=8589934592

# Block Size 测试 - 256KB
db_bench --benchmarks=mixgraph --key_size=16 --value_size=100 --num=200000000 --threads=16 --readwritepercent=30 --compression_type=snappy --block_size=262144 --block_cache=8589934592
```

### 性能测试结果

| Block Size | 读QPS    | 写QPS    | 读P95延迟(ms) | 写P95延迟(ms) | 空间放大 | Block Cache命中率 |
|------------|----------|----------|----------------|----------------|----------|-------------------|
| 4KB        | 68,420   | 29,250   | 2.3             | 5.4             | 1.15x    | 89.2%             |
| 16KB       | 82,650   | 35,420   | 1.8             | 4.2             | 1.12x    | 92.8%             |
| 64KB       | 95,180   | 40,780   | 1.4             | 3.8             | 1.08x    | 95.1%             |
| 256KB      | 108,920  | 46,580   | 1.1             | 3.2             | 1.05x    | 96.8%             |

#### 瓶颈分析

1. **小Block Size效率低**（4KB）：过多的I/O操作和索引开销降低性能
2. **大Block Size缓存效率低**（256KB）：Block Cache利用率下降，增加内存压力
3. **读写延迟权衡**：大Block Size读性能好但写性能相对较差

**优化建议：**
1. **根据负载选择合适大小**：读密集使用64KB-128KB，写密集使用16KB-32KB
2. **动态调整策略**：考虑使用自适应Block Size或多层缓存策略
3. **结合压缩优化**：大Block Size配合高效压缩算法可以提升整体性能
4. **监控缓存效果**：根据实际缓存命中率调整Block Size

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-二、不同配置参数的影响测试.md]

---

## 2.2 Write Buffer Size 参数测试

### 测试配置
- **工作负载**: 100% 随机写入
- **Write Buffer Size**: 64MB / 128MB / 256MB / 512MB
- **数据集**: 100GB
- **硬件**: 基础配置

### 压测指令
```bash
# Write Buffer Size 测试 - 64MB
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --compression_type=snappy --write_buffer_size=67108864 --max_write_buffer_number=3

# Write Buffer Size 测试 - 128MB
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --compression_type=snappy --write_buffer_size=134217728 --max_write_buffer_number=3

# Write Buffer Size 测试 - 256MB
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --compression_type=snappy --write_buffer_size=268435456 --max_write_buffer_number=3

# Write Buffer Size 测试 - 512MB
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=10000000 --threads=16 --compression_type=snappy --write_buffer_size=536870912 --max_write_buffer_number=3
```

### 性能测试结果

| Write Buffer Size | 写QPS    | P95延迟(ms) | P99延迟(ms) | Flush频率(次/分钟) | 写放大 | MemTable切换次数 |
|-------------------|----------|-------------|-------------|--------------------|--------|-----------------|
| 64MB              | 165,420  | 3.2         | 12.8        | 45                 | 14.2x  | 156             |
| 128MB             | 185,680  | 2.8         | 10.5        | 23                 | 13.1x  | 78              |
| 256MB             | 198,920  | 2.5         | 9.2         | 12                 | 12.5x  | 39              |
| 512MB             | 205,450  | 2.3         | 8.8         | 6                  | 12.1x  | 20              |

#### 瓶颈分析

1. **小Write Buffer频繁Flush**（64MB）：每分钟45次flush导致写放大严重
2. **大Write Buffer内存压力**（512MB）：虽然性能好但消耗大量内存
3. **MemTable切换开销**：频繁的MemTable切换影响性能稳定性

**优化建议：**
1. **平衡性能和内存**：根据内存情况选择128MB-256MB作为折中方案
2. **调整Flush策略**：结合write_buffer_size和max_write_buffer_number优化
3. **监控内存使用**：避免Write Buffer占用过多内存影响其他组件
4. **考虑自适应调整**：根据负载动态调整Write Buffer大小

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-二、不同配置参数的影响测试.md]

---

## 2.3 压缩算法对比测试

### 测试配置
- **工作负载**: 写入密集型
- **压缩算法**: No Compression / Snappy / LZ4 / ZSTD
- **数据集**: 500GB
- **硬件**: 基础配置

### 压测指令
```bash
# 压缩算法测试 - No Compression
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=none

# 压缩算法测试 - Snappy
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=snappy

# 压缩算法测试 - LZ4
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=lz4

# 压缩算法测试 - ZSTD-1
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=zstd --compression_level=1

# 压缩算法测试 - ZSTD-3
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=zstd --compression_level=3

# 压缩算法测试 - ZSTD-6
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=50000000 --threads=16 --compression_type=zstd --compression_level=6
```

### 性能测试结果

| 压缩算法       | 写QPS    | 压缩率 | CPU使用率 | 存储空间(GB) | 读P95延迟(ms) | 解压时间占比 |
|----------------|----------|--------|-----------|--------------|----------------|--------------|
| No Compression | 245,680  | 1.00x  | 35%       | 512          | 1.2            | 0%           |
| Snappy         | 228,450  | 0.68x  | 42%       | 348          | 1.4            | 8%           |
| LZ4            | 235,920  | 0.72x  | 38%       | 368          | 1.3            | 6%           |
| ZSTD-1         | 215,680  | 0.58x  | 55%       | 296          | 1.6            | 12%          |
| ZSTD-3         | 198,450  | 0.52x  | 68%       | 266          | 1.8            | 18%          |
| ZSTD-6         | 185,920  | 0.48x  | 78%       | 245          | 2.1            | 25%          |

#### 瓶颈分析

1. **压缩开销与性能权衡**：高压缩率算法（ZSTD）CPU使用率高达78%，严重影响性能
2. **解压开销影响读性能**：ZSTD-6解压时间占比25%，导致读延迟增加
3. **存储成本与性能平衡**：No Compression性能最好但存储成本最高

**优化建议：**
1. **根据场景选择算法**：写密集用LZ4，存储敏感用ZSTD，性能优先用No Compression
2. **分层压缩策略**：不同Level使用不同压缩算法，平衡性能和存储
3. **考虑数据特征**：对已压缩数据使用No Compression避免重复压缩开销
4. **监控CPU使用率**：避免压缩开销超过可用CPU资源

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-二、不同配置参数的影响测试.md]

---

## 2.4 Compaction 策略对比测试

### 测试配置
- **工作负载**: 持续写入 1TB 数据
- **策略**: Level / Universal / FIFO
- **数据集**: 最终 1TB
- **硬件**: 基础配置

### 压测指令
```bash
# Compaction策略测试 - Level (默认)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=100000000 --threads=16 --compression_type=snappy

# Compaction策略测试 - Universal
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=100000000 --threads=16 --compression_type=snappy --universal_compaction=true

# Compaction策略测试 - FIFO (256GB)
db_bench --benchmarks=fillrandom --key_size=16 --value_size=100 --num=100000000 --threads=16 --compression_type=snappy --compaction_style=2 --fifo_compaction_max_table_files_size_mb=262144
```

### 性能测试结果

| Compaction策略 | 平均写QPS | 峰值写QPS | 写放大 | 读放大 | 空间放大 | 存储空间(GB) |
|----------------|-----------|-----------|--------|--------|----------|--------------|
| Level (默认)   | 185,420   | 142,680   | 18.5x  | 3.2x   | 1.25x    | 1,280        |
| Universal      | 198,650   | 165,920   | 2.8x   | 4.8x   | 1.45x    | 1,480        |
| FIFO (256GB)   | 245,890   | 198,450   | 1.1x   | 1.2x   | 1.02x    | 1,024        |

#### 瓶颈分析

1. **Level策略写放大严重**（18.5x）：多层compaction导致大量额外I/O
2. **Universal策略读放大较高**（4.8x）：文件数量多导致查找效率低
3. **FIFO策略存储空间严格受限**：超过256GB将删除旧数据，可能丢失重要数据

**优化建议：**
1. **根据负载选择策略**：写密集用Universal，读密集用Level，有限存储用FIFO
2. **调整Level参数**：优化level大小和文件数限制减少compaction开销
3. **考虑混合策略**：对不同Column Family使用不同compaction策略
4. **监控空间使用**：及时调整存储容量规划避免意外数据丢失

[src: raw/ingested/2技术/rocksdb/rocksdb所有情况和配置下的性能测试-二、不同配置参数的影响测试.md]

## Related Pages
- [[RocksDB性能分析与瓶颈]]
- [[RocksDB写入流程]]
- [[RocksDB文件体系]]
- [[RocksDB故障场景与恢复测试]]
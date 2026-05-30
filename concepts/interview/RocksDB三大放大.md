# RocksDB 三大放大问题分析

## 读放大（Read Amplification）

**定义**：读取数据量 > 实际数据量

**原因**：
- L0 需要遍历所有文件（无序）
- 多层级查询（MemTable → L0 → L1 → ...）
- Compaction 过程中读取多个文件

**优化策略**：
- **Bloom Filter**：快速过滤不存在 Key，减少无效 IO
- **Block Cache**：缓存热数据块，减少磁盘读取
- **减少 L0 文件数量**：及时 Compaction
- **启用分区索引**（Partitioned Index）：7.x+ 特性，减少 Swap 竞争

**监控指标**：
- Block Cache 命中率（建议 > 90%）
- L0 文件数量
- 读取延迟（P50/P95/P99）

## 写放大（Write Amplification）

**定义**：写入数据量 > 实际数据量

**原因**：
- MemTable 刷盘（1 倍）
- Compaction 重写数据（5-10 倍）
- WAL 写入（1 倍）

**优化策略**：
- **Universal Compaction 策略**：减少写放大（2-3 倍）
- **减少 Compaction 频率**：调整触发条件
- **使用轻量级压缩算法**：LZ4（减少 Compaction 数据量）
- **7.x+ 优化**：修复 Compaction 阻塞问题，减少无效重写

**监控指标**：
- Write Amplification Ratio
- Compaction IO 速率
- Write Stall 次数

## 空间放大（Space Amplification）

**定义**：磁盘占用 > 实际数据大小

**原因**：
- 同一 Key 的多个版本同时存在
- 删除标记（Tombstone）未及时清理
- Compaction 不及时

**优化策略**：
- **Leveled Compaction 策略**：每层 Key 唯一，空间放大低
- **及时 Compaction**：清理 Tombstone
- **动态层级大小调整**：7.x+ 特性，空间效率更稳定

**监控指标**：
- 磁盘使用量
- SST 文件数量
- Tombstone 数量

## 三大放大问题权衡

| 放大类型 | Leveled Compaction | Universal Compaction |
|---------|-------------------|---------------------|
| **读放大** | 低 ✅ | 高 ❌ |
| **写放大** | 高 ❌ | 低 ✅ |
| **空间放大** | 低 ✅ | 高 ❌ |

**选择建议**：
- **读多写少**：Leveled Compaction（默认）
- **写多读少**：Universal Compaction
- **平衡场景**：Leveled Compaction + 优化配置

[src: raw/ingested/2技术/rocksdb/rocksdb分析.md]

## Related Pages
- [[RocksDB Compaction]]
- [[RocksDB读取流程]]
- [[RocksDB写入流程]]
- [[RocksDB性能调优]]
- [[OBS对接RocksDB性能分析]]

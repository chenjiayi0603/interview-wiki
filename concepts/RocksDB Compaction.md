# RocksDB Compaction

## Compaction 核心概念

**Compaction 作用**：
1. **合并冗余数据**：同一 Key 的多个版本合并为最新版本
2. **清理删除标记**：Tombstone 在 Compaction 时物理删除
3. **优化读取路径**：减少文件数量，降低读放大
4. **控制空间放大**：及时清理无效数据

**Compaction 触发条件**：
- **L0 触发**：文件数 > `level0_file_num_compaction_trigger`（默认 4）
- **L1+ 触发**：文件大小 > `max_bytes_for_level_base * multiplier`
- **手动触发**：`CompactRange`

**Compaction 优先级**：
- **kByCompensatedSize**：按补偿大小（考虑删除数据）
- **kOldestLargestSeqFirst**：优先处理最旧的大文件
- **kOldestSmallestSeqFirst**：优先处理最旧的小文件
- **kMinOverlappingRatio**：最小重叠比例（推荐）

## Leveled Compaction 详解

**Leveled Compaction 特点**：
- **逐层合并**：L0 → L1 → L2 → ... → Ln
- **有序不重叠**：每层文件按 Key 范围有序，且不重叠
- **空间效率高**：每层 Key 唯一，空间放大低

**合并过程**：
```
1. 选择 L0 文件（可能多个，有重叠）
2. 选择 L1 文件（与 L0 Key 范围重叠）
3. 多路归并排序
4. 生成新的 L1 文件（有序，不重叠）
5. 删除旧文件
```

**写放大分析**：
- **L0 → L1**：读取 L0 文件 + L1 重叠文件，写入新 L1 文件
- **L1 → L2**：读取 L1 文件 + L2 重叠文件，写入新 L2 文件
- **总体写放大**：5-10 倍（取决于层级深度）

**配置参数**：
```cpp
options.level0_file_num_compaction_trigger = 4;        // L0 触发文件数
options.max_bytes_for_level_base = 256 * 1024 * 1024; // L1 大小上限 (256MB)
options.max_bytes_for_level_multiplier = 10;           // 层级倍数
options.target_file_size_base = 64 * 1024 * 1024;     // 目标文件大小 (64MB)
```

**优势与劣势**：
- ✅ **优势**：读放大低、空间放大低
- ❌ **劣势**：写放大高（5-10 倍）

## Universal Compaction 详解

**Universal Compaction 特点**：
- **全量合并**：合并所有文件，不区分层级
- **写放大低**：2-3 倍，适合写密集型
- **空间放大高**：同一 Key 的多个版本可能同时存在

**合并策略**：
- **Size Ratio 触发**：文件大小比例 > `size_ratio`（默认 1）
- **文件数触发**：文件数 > `max_merge_width`（默认 4294967295）
- **空间放大触发**：空间放大 > `max_size_amplification_percent`（默认 200）

**适用场景**：
- **写密集型**：写入多，读取少
- **时序数据**：数据按时间顺序写入，旧数据很少读取
- **日志系统**：追加写入，很少更新

**优势与劣势**：
- ✅ **优势**：写放大低（2-3 倍）
- ❌ **劣势**：读放大高、空间放大高

## Dynamic Leveled Compaction（7.x+）

**特点**：
- **动态调整**：根据最深层文件大小动态调整每层上限
- **空间效率**：减少无效数据，空间效率更稳定
- **配置**：`level_compaction_dynamic_level_bytes = true`

**优势**：
- 空间效率更稳定
- 减少无效数据
- 自动适应数据规模

## Compaction 性能优化

**并行 Compaction**：
- **Subcompactions**：将大 Compaction 任务拆分为多个子任务并行执行
- **配置**：`max_subcompactions > 1`
- **性能提升**：多核 CPU 利用率提升，Compaction 时间缩短

**Compaction 限流**：
- **soft_pending_compaction_bytes_limit**：软限制，开始限流
- **hard_pending_compaction_bytes_limit**：硬限制，完全停止写入
- **rate_limiter**：限制 Compaction IO 速率

**Compaction 优先级**：
- **kMinOverlappingRatio**：优先处理重叠比例小的文件
- **优势**：减少 Compaction 工作量，提高效率

**7.x+ Compaction 优化**：
- **修复阻塞问题**：修复 Pending Bytes 计算放大，避免过度触发 Write Stall
- **增量 Compaction**：分批次处理数据，降低单次资源占用
- **性能提升**：Compaction 期间写入 QPS 波动 < 5%

## Universal Compaction 与 Leveled Compaction 区别对比

| 特性/对比项                | Universal Compaction                    | Leveled Compaction（Level Compaction）              |
|---------------------------|-----------------------------------------|----------------------------------------------------|
| 主要用途                   | 写密集型、低延迟场景                   | 读密集型、兼容大多数通用场景                        |
| 层级结构                   | 无明确层级，所有 SST 文件并列           | 数据按严格层级（L0~Ln）管理                         |
| 合并策略                   | 多个 SST 文件灵活合并，优先合并小文件   | 按层级将文件并入下一层，保证层内无 key 重叠           |
| 写放大（Write Amplification）| 较低                                    | 相对较高                                           |
| 读放大（Read Amplification）| 较高（可能同一个 key 存在多个文件）      | 较低，一般只会命中 1-2 个文件                       |
| 空间放大（Space Amplification）| 相对较高                                | 较低，数据有序且冗余率低                            |
| 查询性能                   | 查询需要遍历多个文件，延迟略高           | 查询命中个数少，延迟稳定                            |
| Compaction 粒度            | 文件级，合并更灵活，可并行性高           | 层级整体迁移，合并规模较大                          |
| 适用强调                   | 高写入吞吐量、日志型、时序型场景         | 低延迟读、面向事务、空间敏感型场景                   |
| 配置参数                   | `options.compaction_style = kCompactionStyleUniversal` | `options.compaction_style = kCompactionStyleLevel`  |

**总结：**
- Universal Compaction 更加灵活，优点是写放大低、可提升写入吞吐，但缺点是读放大和空间放大会略高。适合写密集、数据热点变化大的场景。
- Leveled Compaction 读写更加均衡，适合读密集、空间利用敏感、追求低延迟读的通用业务场景。

[src: raw/ingested/2技术/rocksdb/rocksdb分析.md]

## Related Pages
- [[RocksDB写入流程]]
- [[RocksDB读取流程]]
- [[RocksDB LSM-Tree]]
- [[RocksDB性能调优]]
- [[OBS对接RocksDB性能分析]]

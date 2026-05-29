# RocksDB 性能分析 - 写优化

## 二、RocksDB 为什么性能这么高

### 2.1 LSM-Tree 架构的写优化本质

RocksDB 基于 LSM-Tree（Log-Structured Merge-Tree），其核心设计哲学是 **将随机写转化为顺序写**：

```
用户写入流程:
  Write Request
       ↓
  [WAL (顺序追加写)] ──→ 磁盘顺序 I/O，持久化保障
       ↓
  [MemTable (内存跳表)] ──→ 内存操作，微秒级延迟
       ↓ (MemTable 满)
  [Immutable MemTable] ──→ 冻结，异步刷盘
       ↓ (Flush)
  [Level-0 SST 文件] ──→ 顺序写磁盘
       ↓ (Compaction)
  [Level-1...Level-N SST] ──→ 后台归并排序
```

**为什么这比 B-Tree 快：**

| 维度 | B-Tree (InnoDB) | LSM-Tree (RocksDB) |
|------|----------------|-------------------|
| 写 I/O 模式 | **随机写**（就地更新页面） | **顺序写**（追加 WAL + Flush SST） |
| 单次写 I/O 次数 | 2~4 次（日志+数据页+可能的页分裂） | 1 次（WAL 追加） |
| 写放大 | 10~30×（页面部分更新+日志双写） | 10~30×（Compaction，但分摊到后台） |
| 写入延迟 | 受限于 fsync 随机页 | 仅 WAL fsync（顺序 I/O） |
| SSD 友好度 | 差（随机小写触发 GC） | 极好（大块顺序写入） |

在 NVMe SSD 上，顺序写带宽通常是随机写的 **3~10 倍**，因此 LSM-Tree 的写入路径天然适配现代闪存存储。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-二、RocksDB-为什么性能这么高.md]

### 2.2 内存中的高效写缓冲

RocksDB 使用 **SkipList** 作为默认 MemTable 实现：

- **无锁并发读**：SkipList 支持单写多读的无锁并发访问
- **O(log N) 插入**：内存操作延迟在 **0.5~2 μs**
- **有序性维护**：插入时自动排序，Flush 时直接输出有序 SST
- **内存预分配**：Arena 分配器减少 malloc 碎片，提升 CPU cache 命中率

对比 B-Tree 的页面管理开销（锁页、脏页追踪、buffer pool LRU），SkipList 的内存操作效率高出 **2~5 倍**。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-二、RocksDB-为什么性能这么高.md]

### 2.3 Block Cache + Bloom Filter 的读优化

**Block Cache（LRU / HyperClock）**：
- 缓存热点 SST Block，避免重复磁盘读取
- HyperClock Cache（RocksDB 8.x+）使用无锁时钟算法，在高并发下比 LRU 快 **10~30%**
- 典型配置 6~16GB Cache，可覆盖工作集的 50%~80%

**Bloom Filter**：
- 每个 SST 文件维护 Bloom Filter（通常 10 bits/key）
- 点查询时先检查 Bloom Filter，**误判率 ~1%**
- 避免了 90%+ 的无效磁盘读取
- 内存开销：100M keys 仅需 ~125MB

**多级缓存体系**：
```
查询路径:
  MemTable (内存, ~μs)
    ↓ miss
  Immutable MemTable (内存, ~μs)
    ↓ miss
  Block Cache (内存, ~μs)
    ↓ miss
  Bloom Filter 检查 (内存, ~ns)
    ↓ 可能存在
  SST 文件读取 (磁盘, ~100μs NVMe / ~ms HDD)
```

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-二、RocksDB-为什么性能这么高.md]

### 2.4 后台 Compaction 分摊写成本

Compaction 是 LSM-Tree 的核心维护操作，RocksDB 将其完全推到后台线程：

- **用户写入只负责 WAL + MemTable**，微秒级返回
- **Compaction 在后台异步执行**，使用独立线程池（默认 `max_background_jobs=2`）
- **CPU 密集型排序与 I/O 重叠**：Compaction 读取旧 SST → 归并排序 → 写入新 SST，可并行于用户读写
- **分层 Compaction 策略**：Level Compaction 每层数据量 10× 增长，确保读放大 ≤ level 数（通常 5~7 层）

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-二、RocksDB-为什么性能这么高.md]

### 2.5 批量写与 Group Commit

RocksDB 通过 **WriteBatch** 和 **Group Commit** 进一步减少写 I/O：

- **WriteBatch**：将多个 Put/Delete 操作打包成一个原子写，减少 WAL fsync 次数
- **Group Commit**：当多个线程同时提交写入时，Leader 线程将所有 pending 写入合并成一次 WAL fsync
- 单次 fsync 的开销约 **50~200 μs**（NVMe），Group Commit 可将 N 次写入分摊到 1 次 fsync

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-二、RocksDB-为什么性能这么高.md]
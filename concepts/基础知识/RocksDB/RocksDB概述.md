# RocksDB 概述

## 定位与特点

- **定位**：Facebook 开源的高性能嵌入式键值存储引擎，基于 LSM-Tree 架构
- **核心特点**：
  - **高吞吐写入**：顺序写入优化，写入性能是 B+ 树的 10-100 倍
  - **低延迟读取**：内存查询 + Bloom Filter + Block Cache
  - **嵌入式设计**：以库形式集成，无需独立服务进程
  - **高度可配置**：支持列族、多种 Compaction 策略、压缩算法
  - **生产级特性**：事务支持、快照、备份恢复

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 核心架构组件

| 组件 | 作用 | 关键特性 |
|-----|------|---------|
| **WAL** | 预写日志，保证持久性 | 顺序写入，崩溃恢复 |
| **MemTable** | 内存跳表，接受写入 | 写满后转为 Immutable |
| **Immutable MemTable** | 只读内存表，等待刷盘 | 后台 Flush 生成 L0 SST |
| **SST 文件** | 磁盘有序文件 | L0 无序，L1+ 有序且不重叠 |
| **Block Cache** | 热数据缓存 | LRU 策略，减少磁盘 IO |
| **Bloom Filter** | 快速过滤不存在 Key | 10 bits/key 典型配置 |

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 三大放大问题

| 放大类型 | 定义 | 优化策略 |
|---------|------|---------|
| **读放大** | 读取数据量 > 实际数据量 | Bloom Filter、Block Cache、减少层级 |
| **写放大** | 写入数据量 > 实际数据量 | Universal Compaction、减少 Compaction 频率 |
| **空间放大** | 磁盘占用 > 实际数据大小 | Leveled Compaction、及时清理 Tombstone |

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 写入流程速记

```
写入请求 → WAL（预写日志）→ MemTable（内存跳表）
→ MemTable 满 → Immutable MemTable → Flush → L0 SST
→ Compaction → L1-Ln SST
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 读取流程速记

```
查询请求 → MemTable → Immutable MemTable → L0 SST（遍历所有）
→ L1+ SST（二分查找）→ Block Cache → Bloom Filter 过滤
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 生产推荐配置

- **版本**：RocksDB 7.5.3+（解决 Compaction 阻塞问题）
- **Compaction**：Leveled + Dynamic Level Bytes
- **压缩**：L0 禁用，底层 Zstd
- **内存**：Block Cache = 总内存 1/3，启用分区索引
- **并发**：max_background_jobs = 6，启用并行 Compaction

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Related Pages
- [[RocksDB LSM-Tree]]
- [[RocksDB写入流程]]
- [[RocksDB读取流程]]
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]
- [[RocksDB列族与事务]]
- [[RocksDB监控与故障排查]]
- [[RocksDB实际应用案例]]
- [[RocksDB版本演进]]
- [[RocksDB面试考点]]

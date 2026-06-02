# RocksDB 总览

> RocksDB 高性能嵌入式键值存储引擎知识体系：LSM-Tree 架构、读写流程、文件格式、线程模型、性能分析与面试考点。

---

## 一、文件地图

| 文件 | 内容 |
|------|------|
| [00-rocksdb-总览.md](00-rocksdb-总览.md) | 本文件：索引 + 知识体系全景 |
| [01-rocksdb-LSM架构与核心原理.md](01-rocksdb-LSM架构与核心原理.md) | LSM-Tree 架构、核心组件、读写流程、Compaction 策略 |
| [02-rocksdb-文件格式.md](02-rocksdb-文件格式.md) | SST / WAL / MANIFEST / CURRENT 等文件结构详解 |
| [03-rocksdb-线程模型.md](03-rocksdb-线程模型.md) | 线程架构、Leader-Follower 模型、并发控制 |
| [04-rocksdb-性能分析与调优.md](04-rocksdb-性能分析与调优.md) | 三大放大、性能基准、瓶颈原因与优化手段 |
| [05-rocksdb-面试考点速查.md](05-rocksdb-面试考点速查.md) | 面试高频 Q&A、KeyDB 存算分离项目 |

---

## 二、知识体系全景

```
┌──────────────────────────────────────────────────────────────┐
│                    RocksDB 知识体系                             │
├──────────────────────────────────────────────────────────────┤
│  核心架构                   写入路径                            │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ LSM-Tree         │    │ WriteBatch       │                  │
│  │ MemTable/SkipList │    │ WAL (预写日志)    │                  │
│  │ SST (L0~Ln)      │    │ Leader-Follower   │                  │
│  │ Block Cache      │    │ Flush → L0       │                  │
│  │ Bloom Filter     │    │ Compaction       │                  │
│  └──────────────────┘    └──────────────────┘                  │
├──────────────────────────────────────────────────────────────┤
│  文件格式与线程            性能与调优                            │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ SST 内部结构      │    │ 写放大 (5-10x)    │                  │
│  │ WAL 格式         │    │ 读放大 (多层查询)  │                  │
│  │ MANIFEST/CURRENT │    │ 空间放大          │                  │
│  │ 线程池模型        │    │ Write Stall      │                  │
│  │ Env 插件机制      │    │ 内存/压缩优化     │                  │
│  └──────────────────┘    └──────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 三、核心组件全景

| 组件 | 位置 | 数据结构 | 作用 | 关键参数 |
|-----|------|---------|------|---------|
| **WAL** | 磁盘 | 顺序日志 | 预写日志，保证持久性 | `sync_log`, `max_total_wal_size` |
| **MemTable** | 内存 | SkipList | 接受写入，O(log n) 插入/查询 | `write_buffer_size` (默认 64MB) |
| **Immutable MemTable** | 内存 | SkipList | 只读，等待刷盘 | `max_write_buffer_number` (默认 2) |
| **L0 SST** | 磁盘 | 无序 SST | Flush 生成，文件间可重叠 | `level0_file_num_compaction_trigger` (默认 4) |
| **L1-Ln SST** | 磁盘 | 有序 SST | 按 Key 范围有序，不重叠 | `max_bytes_for_level_base` (默认 256MB) |
| **Block Cache** | 内存 | LRU Cache | 缓存热数据块，减少磁盘 IO | `block_cache_size` (建议总内存 1/3) |
| **Bloom Filter** | 内存/磁盘 | 位数组 | 快速过滤不存在 Key | `bits_per_key` (默认 10) |

---

## 四、典型应用场景

| 应用场景 | 代表系统 | 使用优势 |
|---------|---------|---------|
| 分布式数据库存储引擎 | TiDB (TiKV) | 高吞吐写入、事务支持、多列族 |
| 流计算状态后端 | Flink StateBackend | 大状态存储、增量 Checkpoint |
| 消息队列持久化 | Kafka | 顺序写入优化、高吞吐 |
| 时序数据库 | InfluxDB | 高效追加写入、时间范围查询 |
| 缓存系统持久化 | Pika (Redis 协议) | 内存+磁盘混合存储 |
| 对象存储元数据 | Ceph BlueStore | 嵌入式、高性能元数据操作 |

# RocksDB LSM-Tree

## LSM-Tree 设计思想

**核心思想**：
- **写入优化**：将随机写转换为顺序写，利用磁盘顺序写入性能优势（顺序 IO 比随机 IO 快 10-100 倍）
- **分层存储**：热数据在内存，冷数据在磁盘，自动分层管理
- **延迟合并**：写入时只追加，Compaction 时合并，分摊写入成本

**与 B+ 树对比**：

| 维度 | B+ 树 | LSM-Tree (RocksDB) |
|-----|------|-------------------|
| **写入方式** | 随机写入（原地更新） | 顺序写入（追加） |
| **写入性能** | 低（随机 IO，受磁盘寻道时间限制） | 高（顺序 IO，10-100 倍） |
| **读取性能** | 高（单次查询，O(log n)） | 中等（可能多级查询，但优化后接近） |
| **空间放大** | 低（无冗余，原地更新） | 中等（Compaction 前有冗余） |
| **写放大** | 低（1 倍，原地更新） | 高（5-10 倍，Compaction 重写） |
| **适用场景** | 读多写少（OLTP） | 写多读少（时序、日志） |

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 核心架构组件

```
┌─────────────────────────────────────────────────────────┐
│                    写入路径                              │
│  WriteBatch → WAL → MemTable → Immutable → L0 SST      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                  Compaction 路径                         │
│              L0 → L1 → L2 → ... → Ln                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                    读取路径                              │
│  MemTable → Immutable → L0 → L1+ → Block Cache         │
└─────────────────────────────────────────────────────────┘
```

**核心组件详解**：

| 组件 | 位置 | 数据结构 | 作用 | 关键参数 |
|-----|------|---------|------|---------|
| **WAL** | 磁盘 | 顺序日志 | 预写日志，保证持久性，崩溃恢复 | `sync_log`, `max_total_wal_size` |
| **MemTable** | 内存 | SkipList（跳表） | 接受写入操作，O(log n) 插入和查询 | `write_buffer_size` (默认 64MB) |
| **Immutable MemTable** | 内存 | SkipList | 只读，等待刷盘，支持并发查询 | `max_write_buffer_number` (默认 2) |
| **L0 SST** | 磁盘 | 无序 SST | Flush 生成，可能重叠，需要遍历 | `level0_file_num_compaction_trigger` (默认 4) |
| **L1-Ln SST** | 磁盘 | 有序 SST | 按 Key 范围有序，不重叠，二分查找 | `max_bytes_for_level_base` (默认 256MB) |
| **Block Cache** | 内存 | LRU Cache | 缓存热数据块，减少磁盘 IO | `block_cache_size` (建议总内存 1/3) |
| **Bloom Filter** | 内存/磁盘 | 位数组 | 快速过滤不存在 Key，减少无效 IO | `bits_per_key` (默认 10) |

**关键点**：
- MemTable 写满后转为 Immutable MemTable，创建新 MemTable 继续写入
- Immutable MemTable 由后台线程 Flush 生成 L0 SST 文件
- L0 文件无序且可能重叠，L1+ 文件有序且不重叠

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## SST（Sorted String Table）结构说明

SST 全称为 *Sorted String Table*，即"有序字符串表"。它本质上是磁盘上的**有序键值对文件**，通常是持久化的、不可变的数据块（immutable）。

SST 文件内部按照 Key**有序排列**，方便基于范围的查找和高效的合并（Compaction）。SST 通常以块（block）为单位存储，包含如下结构：

- **数据块（data block）：** 实际存放有序的键值对。
- **索引块（index block）：** 按 key 范围建立索引，支持跳跃/二分查找。
- **元数据块（meta block/filter block）：** 存放元数据信息、Bloom Filter 等辅助数据。
- **Footer：** 存储整个文件各个 block 的偏移信息。

典型的 SST 结构如下：

```
┌─────────────────────────────────────┐
│  Data Block #1  (key-value...)      │
├─────────────────────────────────────┤
│  Data Block #2                      │
├─────────────────────────────────────┤
│  ...                                │
├─────────────────────────────────────┤
│  Index Block                        │
├─────────────────────────────────────┤
│  Meta Block / Filter Block           │
├─────────────────────────────────────┤
│  Footer (block offset, magic, ...)  │
└─────────────────────────────────────┘
```

**优点：**
- 有序：支持二分查找和范围查找。
- 只追加、不变更：便于多路归并和高并发读取。
- 支持 Bloom Filter，过滤不命中的请求。

**在 RocksDB 中，MemTable flush 到磁盘后就生成一个新的 SST 文件；多级 Compaction 过程也是不停地生成和合并新的 SST 文件。**

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## LSM-Tree 设计原理及应用

**LSM-Tree（日志结构合并树）** 是一种高度写优化的存储引擎结构，最早由 Patrick O'Neil 等人在1996年提出，旨在应对以写为主、高吞吐、低延迟的存储需求。其核心思想是：
- **所有写入操作首先顺序写到内存（MemTable）和追加式日志（WAL）中**，而不是直接修改磁盘文件。
- **内存结构写满后批量顺序刷盘，生成新的有序磁盘文件（SST/表格文件）**，此时依然不会做随机写。
- **磁盘文件采用多级分层结构**（L0~Ln），新文件永远是追加写，老文件在后台通过合并（Compaction）将数据归并到更低层，同时清除孤儿、历史或被覆写的数据，确保空间和读的高效。

**为什么高效？**
- 极度优化写操作：全局顺序追加、内存聚集、不做随机磁盘写，吞吐极高。
- 读虽需多层查找，但可以用索引、Bloom Filter等结构高效过滤，并用缓存加速热点数据。
- 后台合并/清理可异步平滑执行，前台操作快，影响极小。

**工业界的 LSM-Tree 应用举例**

LSM-Tree 不仅仅用于 RocksDB，实际上它已成为**现代写优化型存储引擎的事实标准**，广泛应用于许多著名工业数据库、分布式系统和云原生产品中。例如：

- **LevelDB**：Google 开源的嵌入式KV存储引擎，最早广泛应用 LSM-Tree。
- **RocksDB**：由 Facebook 基于 LevelDB 的设计深度优化扩展，成为工业界 LSM-Tree 的标杆实现。
- **HBase**、**Cassandra**、**ScyllaDB**：大数据 NoSQL 数据库，底层全部采用 LSM-Tree 设计，实现超高可扩展性和强大写入吞吐。
- **ClickHouse MergeTree 引擎家族**、**TiKV**（TiDB 存储层）等新一代分布式存储引擎，都深度借鉴或直接采用 LSM-Tree 思路。
- **InnoDB（部分）**、**TimescaleDB**、**InfluxDB** 这样的时间序列、日志型或 OLAP 场景中，也常见 LSM-Tree 或其变种作为存储、索引方案。

**小结**

LSM-Tree 的设计本质解决了写多于读、以及"写瓶颈"场景下存储效率问题，是支撑今日大部分高性能 KV 数据库、日志型、时间序列和分布式存储系统的底层核心架构之一。其思想已深入渗透到工业界众多数据库与现代云原生基础设施中。

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## RocksDB 对 LSM-Tree 的工程实现与优化

RocksDB 将 LSM-Tree 体系发挥到了工业级水平，具体体现在：

1. **数据写入分层管理**
   - 所有写操作先进入 WAL 保证持久性，再到 MemTable（基于高效的 SkipList 为主结构）。
   - MemTable 满后自动转为不可变 Immutable MemTable，由后台线程顺序落盘为有序的 L0 SST 文件。

2. **多级磁盘文件体系（L0~Ln）**
   - L0 文件无序、可重叠，来源于 MemTable 的直接 Flush。
   - L1 及以上层级严格有序、不可重叠，提升范围查询和 Compaction 效率。
   - 多级 Compaction 策略（Level、Universal 等）灵活平衡写放大、空间放大和读写延迟。

3. **Compaction 的高效实现**
   - RocksDB 精细调度 Compaction（分层式、全量式、单列族/多列族并发等），后台线程异步多进程，规避前台堵塞。
   - 支持 Rate Limiter、动态合并粒度、回收过期/冗余数据，减轻写放大与冷层膨胀。

4. **查询与过滤优化**
   - 读路径充分利用内存的 MemTable、Immutable MemTable、BlockCache（LRU）和每个 SST 内置的 BloomFilter，快速判定 Key 是否可能存在，大幅减少无谓的磁盘 IO。
   - SST 有序，高效支持范围查询和多层合并读取。

5. **并发和容错能力**
   - 多线程并行 Flush、Compaction，Leader-Follower 写入模式提升多核写吞吐。
   - WAL（预写日志）和 Manifest（元数据日志）等机制共同保证了崩溃恢复的可靠性和原子性。
   - Manifest 的作用：记录整个 RocksDB 数据库所有 SST 文件的元数据、版本、层级、范围、compaction 变更历史等，是数据库状态的"总账本"。
   - Manifest 主要保存：
     - 所有现存 SST 文件的文件名、所属层级（Level）、key 范围、文件大小等
     - 最新的 Version 结构信息（代表当前数据库快照）
     - 历次元数据变更的操作日志（如新增、删除文件等）
     - 配合 CURRENT 文件实现崩溃后快速还原数据库结构和文件集，确保每次重启都能完整恢复到一致状态

6. **拓展性与可配置性**
   - 参数高度可调（如 write_buffer_size、max_background_jobs、block_cache_size 等），自适应不同业务场景和硬件资源。
   - 支持多列族、快照、事务、备份恢复等企业级功能，无缝支持大规模分布式系统的状态后端需求。

**小结：**
LSM-Tree 架构赋予 RocksDB 极强的可扩展性和写入性能，是高吞吐/低延迟场景下主流的存储内核方案。RocksDB 在原生 LSM-Tree 的基础上，通过灵活的 Compaction 策略、内存/持久化层协作、多种优化机制，让其在大数据和分布式系统工程中获得广泛应用。从写入、落盘、归并、读取到多副本/恢复，LSM-Tree 思路贯穿 RocksDB 的每一个关键环节。

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Related Pages
- [[RocksDB概述]]
- [[RocksDB写入流程]]
- [[RocksDB读取流程]]
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]
- [[OBS对接RocksDB性能分析]]

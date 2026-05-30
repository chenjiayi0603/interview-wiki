# RocksDB 读取流程

## 完整读取路径

```
1. 获取 SuperVersion（当前数据版本快照）
   ↓
2. 查询 MemTable（跳表，O(log n)）
   ↓
3. 查询 Immutable MemTable
   ↓
4. 查询 L0 SST（遍历所有文件，可能重叠）
   - 使用 FileMetaData 快速判断 Key 范围
   - 按 Sequence Number 从新到旧
   ↓
5. 查询 L1+ SST（每层仅查一个文件）
   - 使用 FileMetaData 二分查找定位文件
   - 使用 Index Block 定位数据块
   - 使用 Bloom Filter 快速过滤
   ↓
6. 查询 Block Cache（LRU 策略）
   ↓
7. 从磁盘读取数据块
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## SuperVersion 机制

**SuperVersion 作用**：
- **版本快照**：记录当前时刻所有 SST 文件列表
- **引用计数**：防止查询过程中文件被清理
- **原子切换**：Compaction 完成后原子更新 SuperVersion

**SuperVersion 组成**：
- MemTable 列表
- Immutable MemTable 列表
- 各级 SST 文件列表（Version）
- Column Family 配置

**使用流程**：
```
1. 获取 SuperVersion（引用计数+1）
2. 执行查询操作
3. 释放 SuperVersion（引用计数-1）
4. 引用计数为 0 时清理旧版本
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## 多级查询策略

**查询顺序**：
```
MemTable → Immutable MemTable → L0 SST → L1 SST → ... → Ln SST
```

**L0 查询特点**：
- **无序性**：文件无序，可能重叠
- **策略**：遍历所有文件，按 Sequence Number 从新到旧
- **优化**：使用 FileMetaData 快速判断 Key 范围

**L1+ 查询特点**：
- **有序性**：文件有序，不重叠
- **策略**：二分查找定位文件，每层只需查询一个文件
- **优化**：FileMetaData 存储 min_key 和 max_key，快速过滤

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## SST 文件查询优化

**SST 文件结构**：
```
[File Header]
[Data Block 1]
[Data Block 2]
...
[Data Block N]
[Index Block]      ← 数据块索引
[Filter Block]     ← Bloom Filter
[Meta Block]       ← 元数据
[Footer]           ← 文件尾部，包含索引位置
```

**文件定位流程**：
1. **FileMetaData 查询**：通过 min_key 和 max_key 判断 Key 是否在文件范围内
2. **Index Block 查询**：二分查找定位 Data Block
3. **Data Block 查询**：在 Block 内查找 Key

**Bloom Filter 优化**：
- **快速过滤**：查询前先查 Bloom Filter
- **不存在判断**：返回"不存在"则跳过该文件
- **可能存在**：返回"可能存在"则继续查询
- **配置**：`bits_per_key`（默认 10），越大精度越高但内存占用越大

**Block Cache 优化**：
- **缓存策略**：LRU（Least Recently Used）
- **缓存内容**：Data Block、Index Block、Filter Block
- **配置**：`block_cache_size`（建议总内存 1/3）
- **命中率监控**：低于 90% 需扩容

**Block Cache 的作用**：

Block Cache（块缓存）是 RocksDB 读取性能的核心优化之一，其主要作用是缓存最近读取过的 Data Block、Index Block 和 Filter Block 等数据块，减少对磁盘的随机读取，从而大幅提升读操作的速度。其主要作用可以归纳如下：

- **加速读请求**：将频繁访问的块缓存在内存中，命中 Block Cache 时无需访问磁盘，极大地提升了查询性能。
- **降低 IO 压力**：缓存热点数据后，减少了对底层存储设备的访问频率，延长了磁盘寿命并降低了系统整体延迟。
- **支持多种类型的 Block**：不仅缓存用户数据块（Data Block），也缓存索引块（Index Block）和 Bloom Filter 块（Filter Block），使目录结构等元数据查找同样受益。
- **内存管理灵活**：通常采用 LRU（最近最少使用）或自适应策略，自动剔除冷数据，始终保持缓存中存放的是近期最热的数据块。
- **可配置**：通过 `block_cache_size` 参数灵活调整内存占用，有效利用服务器内存资源，达到最优读吞吐量。
- **命中率可监控**：通过指标监控命中率，如果发现命中率较低可以提示运维人员扩容缓存，进一步提高性能。

RocksDB 默认建议 Block Cache 占用总内存的 1/3~1/2。在高并发读多写少的场景下，合理设置 Block Cache 大小对于整库的吞吐量具有极其重要的意义。

在 RocksDB 的 SST 文件中，Data Block、Index Block 和 Filter Block 分别代表不同的数据结构：
- **Data Block**：存储实际的用户 Key-Value 数据，查询时要定位到具体的 Data Block 才能读取目标 Key 的值；
- **Index Block**：存储每个 Data Block 的索引信息（如首个 Key 到 Data Block 偏移），用于加速定位 Data Block，减少遍历；
- **Filter Block**：通常为 Bloom Filter，实现快速判断某 Key 是否可能在本文件，能快速排除大部分不存在的 Key，显著减少无效 IO。

**为什么要缓存它们？**

因为每次读操作都可能频繁访问这些 Block，如果每次都从磁盘读取（即便是 SSD），也会有额外延迟和 IO 压力，影响系统吞吐。将 Data Block、Index Block、Filter Block 缓存在 Block Cache（块缓存）中，能大幅提升命中率：
- 查询命中缓存，不需 disk IO，速度极快；
- 热 Key/热点区块多次访问时利用内存复用，减少对底层存储的访问压力；
- Index Block 和 Filter Block 被频繁用于加速查找，如果不缓存，每次查询都需从磁盘加载，性能损失显著。

因此，缓存这三类 Block 可以显著提升 RocksDB 的读性能，是其最关键的优化手段之一。

Block Cache（块缓存）为什么不直接缓存整个 SST 文件？主要原因如下：

1. **SST 文件通常较大，超出内存承受范围**
   单个 SST 文件可能几十 MB 到几百 MB，一个 RocksDB 实例往往有数百个甚至上千个 SST 文件。如果直接将每个文件完整加载到内存，内存消耗极大，远超大多数服务器的可用内存（比如 128GB 的 RocksDB，每个 SST 64MB，1000 个 SST 文件就 64GB 了，实际生产环境中 SST 文件总量通常更大）。

2. **实际访问存在热点与冷数据**
   读操作具有明显的局部性：大部分读请求集中在很少一部分的 Key、Block 上。将全量 SST 文件加载到内存其实浪费了大量内存去缓存冷数据（很少甚至从未被访问）。而 Block Cache 只缓存被频繁访问、热点的 Block，内存命中率更高，资源利用率更优。

3. **分块能更灵活管理内存淘汰和调度**
   Block LRU/Replacer 等算法可以细粒度按 Block 做缓存命中和回收，保证热点数据更久地驻留内存，而冷数据可以及时让出空间。如果直接以整个 SST 为颗粒度，会导致很多不常用的数据长时间霸占内存。

4. **SST 文件设计为磁盘友好存储，块级随机读取高效**
   RocksDB SST File 设计时，充分考虑了磁盘访问特点，Block 对齐、压缩、主索引+Filter Block 极大降低随机 IO 的成本，没必要非得整文件读入——分块+按需加载已经很高效。

5. **缓存整个 SST 会限制吞吐及扩展性**
   若限制为"必须全部 SST 可装入内存"，会极大降低数据库的数据量上限和吞吐能力，影响系统横向扩展能力。

综上，RocksDB 采用"Block Cache"方案——不是缓存整文件，而是按 Block 颗粒度（数据块、索引块、Filter 块等）智能管理热点数据，极大兼顾查询性能和内存资源利用率。这是一种工程上更均衡、更高效的设计。

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Index 与 Filter 结构与原理

### Index（索引）

- Index 是 RocksDB SST 文件的数据块索引，通常为稠密有序的 key-offset 映射表。
- 每个 BlockBased Table 类型的 SST 文件末尾，会写入一个"索引块"（index block）：
    - 其结构本质类似"小型的有序数组/查找表"，存的是每个 data block 的最小 key 及其在文件中的偏移量。
    - 查找流程：先二分 index block 内所有数据块的范围（最快 B+树目录页的角色），定位目标 key 可能位于哪个 data block，再仅读取对应 block 校验内容。
- RocksDB index block 支持存放在 Block Cache 中，极大减少频繁的查表随机小 IO，提升点查性能。
- index 作用：避免全文件遍历，快速定位目标 key 处于哪个数据块（进一步命中 Block Cache 或做磁盘预读）。

```
index 查找流程图（手绘 ASCII 示意）：

       ┌─────────────┐
       │   查找 key  │
       └─────┬───────┘
             │
             v
     ┌───────────────┐
     │  index block  │
     │(key-偏移映射) │
     └─────┬─────────┘
           │
           v
 ┌─────────────────┐
 │ 定位 data block │
 └─────┬───────────┘
       │
       v
 ┌───────────────────┐
 │ 读 data block     │
 └─────┬─────────────┘
       │
       v
 ┌───────────────────┐
 │ 精确查找/比对 key │
 └───────────────────┘
```

流程要点：
1. 先在 index block（通常已缓存）里二分查找找到 key 可能落在哪个 block。
2. 然后只从对应 data block 读取（缓存命中或触发一次磁盘 IO）。
3. 最后在数据块内精确查找目标 key。
（有 filter 时，流程变成 filter → index → data block，filter 先提前淘汰绝大部分不存在的查找。）

下面补充 index block 在整个 RocksDB 数据结构/查询流程中的位置示意图，帮助直观理解其关键作用：

```
RocksDB 查询主流程简图（重点突出 index block 角色）

         ┌────────────┐
         │  上游 GET  │
         │  (用户请求)│
         └─────┬──────┘
               │
        ┌──────▼─────┐
        │ MemTable   │ <─────────┐
        └───▲───┬────┘           │（如在内存，直接命中，跳过磁盘）
            │   │                │
   ┌────────┘   └────────────────┘
   │   （MemTable/L0/L1~LN 层级组成 LSM 树）
   │
   │    SST 文件 (磁盘存储)
   │    ┌──────────────────────────────────────────────┐
   │    │                 SST 文件结构                  │
   │    │                                              │
   │    │ ┌─────────┐   ┌──────────┐   ┌────────────┐  │
   │    │ │ Data    │   │ Index    │   │ Filter     │  │
   │    │ │ Blocks  │   │ Block    │   │ Block      │  │
   │    │ └────┬────┘   └────┬─────┘   └─────┬──────┘  │
   │    │      │             │               │         │
   │    │      │             │               │         │
   │    └──────┼─────────────┼───────────────┼─────────┘
   │           │             │               │
   │           ▼             ▼               ▼
   │     [实际 KEY/VALUE]   [最小key→偏移表] [Bloom位图]
   │
   │
   ├───────────────── 查询流程例子 ──────────────────┐
   │ 1. 先查 Bloom Filter Block（过滤绝大部分无效读）│
   │ 2. 再查 Index Block（Index Block 并不是逐条保存所有 Key，而是每个 Data Block 只在 Index Block 中保留该数据块的"最小 key"（即该块内按排序第一个 key），以及该数据块在文件中的偏移位置，形成有序的映射关系。查找时通过二分法在 Index Block 中快速定位目标 Key 可能所在的 Data Block。这里的"最小/首" key，就是指每个 data block 内排序靠前的那个 key。│
   │ 3. 只读取定位到的 Data Block，在该块内查找目标 Key 不是"顺序遍历逐条比对"，而是利用块内高效的二分查找（如跳表、稀疏索引等，RocksDB 默认采用块内顺序查找+Short Seek 优化），一般只需极少比对就能定位到目标 Key。所以整体读取流程非常高效，不是从头到尾一个个扫描。 │
   │
   └───────────────────────────────────────────────┘

```

简要说明：
- **Filter Block**（Bloom filter）：先用极小开销判断 Key 很可能不在本 SST 文件，大多数无关请求直接排除，避免后续 IO。
- **Index Block**：若 filter 通过，利用 index block 的有序映射快速二分查找，确定 key 所属的 data block（通常 index block 缓存在 block cache，IO 开销极小）。
- **Data Block**：从磁盘或缓存读取目标数据块，最终定位和比对 key。

整体来看，index block 是衔接 filter block（高效排除）和 data block（真正存储数据）之间的有序导航结构，是提升 RocksDB 海量 KV 查询效率的核心"查目录"部件。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

### Filter（Bloom 过滤器）

- filter 在大多数场景指的是 Bloom Filter——一种空间效率极高、允许快速判断"不存在"但不能精确判断"存在"的概率型数据结构。
- RocksDB 按 data/block 粒度为每个 SST 文件（或块）生成 Bloom filter：
    - 结构上就是为文件中所有 key 构造一个 Bloom 位向量，通过 k 次 hash 把 key 映射到若干 bits 上。
    - 读请求流程：先查 Bloom filter，若为"确定不存在"则省去此 SST 的读 IO，只有"可能存在"才去查 index block+data block。
- filter 的作用：快速排除全量大部分一定不存在的 key，显著减少无效磁盘访问（命中概率极低）。
- Bloom filter 误判率由 bits_per_key 决定（如 10~12 位/键典型能做到万分之一误判率）。

总结对比：
- index = 快速定位数据块，适用于查找文件内部 key 具体位置
- filter = 快速排除不在本文件/块，大幅降低无用 IO（提升点查QPS）

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析与瓶颈原因和优化手段.md]

## 前缀查询优化

**前缀 Bloom Filter**：
- **原理**：对 Key 前缀构建 Bloom Filter
- **优势**：范围查询时快速过滤
- **配置**：`prefix_extractor` + `optimize_filters_for_hits`

**Seek 优化**：
- **Next/Prev 优化**：利用 Data Block 有序性，减少查找次数
- **Iterator 缓存**：复用 Iterator，减少创建开销

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Related Pages
- [[RocksDB概述]]
- [[RocksDB LSM-Tree]]
- [[RocksDB写入流程]]
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]
- [[RocksDB性能分析与瓶颈]]
- [[OBS对接RocksDB性能分析]]

# RocksDB 文件体系

本文档从**文件类型、目录布局、各文件格式与作用**等角度，系统梳理 RocksDB 在磁盘上的文件体系，便于理解存储结构、排查问题和面试复习。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

---

## 一、文件类型总览

RocksDB 在数据库目录下会生成以下几类文件：

| 类型 | 典型文件名/后缀 | 作用 |
|------|-----------------|------|
| **WAL** | `*.log` | 预写日志，保证持久性与崩溃恢复 |
| **SST** | `*.sst` | 有序键值数据文件，分层存储（L0~Ln） |
| **MANIFEST** | `MANIFEST-*` | 元数据日志，记录版本与 SST 视图 |
| **CURRENT** | `CURRENT` | 指向当前生效的 MANIFEST 文件 |
| **OPTIONS** | `OPTIONS-*`、`OPTIONS` | 持久化配置（可选） |
| **LOCK** | `LOCK` | 进程级锁，防止多进程同时打开同一 DB |
| **IDENTITY** | `IDENTITY` | 数据库实例唯一标识（可选） |

**列族与共享关系**：

- **所有列族共享**：WAL、CURRENT、MANIFEST、LOCK、OPTIONS
- **每个列族独立**：MemTable、Immutable MemTable、SST 文件（按列族组织）

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

---

## 二、目录布局示意

```
<db_path>/
├── CURRENT              # 当前 MANIFEST 指针
├── LOCK                 # 文件锁
├── IDENTITY             # 实例 ID（可选）
├── OPTIONS-*            # 持久化选项（可选）
├── MANIFEST-000001      # 元数据日志（数字递增）
├── 000003.log            # WAL 文件（数字为 log number）
├── 000005.log
├── 000007.sst            # SST 文件（数字为 file number）
├── 000008.sst
└── ...
```

若配置了独立的 WAL 目录（`wal_dir`），则 WAL 的 `*.log` 会出现在 `wal_dir` 下，其余文件仍在 `db_path`。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

---

## 三、各文件详解

### 3.1 WAL（*.log）

**作用**：

- 在写入 MemTable 之前，先将写操作顺序写入 WAL，保证**持久性**。
- 崩溃恢复时，通过重放 WAL 重建 MemTable，再按需 Flush 成 SST。

**特点**：

- **顺序追加**，写性能好。
- 与 MemTable 一一对应关系：一个 MemTable 对应一段 WAL；MemTable Flush 成 SST 并确认无需该 WAL 后，对应 WAL 可被回收/删除。

**WAL 文件格式（逻辑记录）**：

```
[Record Length: 4 bytes]
[Sequence Number: 8 bytes]
[Record Type: 1 byte]     // PUT / DELETE / MERGE 等
[Key Length: varint]
[Key Data]
[Value Length: varint]    // DELETE 无 Value
[Value Data]
[CRC32: 4 bytes]
```

**切换时机**：

1. WAL 总大小超过 `max_total_wal_size`
2. MemTable 切换（Flush 触发新 MemTable，可能伴随新 WAL）
3. 手动 `FlushWAL()`

**相关配置**：

- `sync_log`：是否每次写 WAL 后同步落盘（true 更安全，false 性能更好）
- `max_total_wal_size`、`wal_dir`

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

### 3.2 SST 文件（*.sst）

**全称**：Sorted String Table，有序字符串表。

**作用**：

- 磁盘上**持久化**的键值数据，按 Key 有序排列。
- 由 Immutable MemTable **Flush** 生成 L0 文件，再由 **Compaction** 产生 L1~Ln 文件。

**层级特性**：

- **L0**：由 Flush 直接生成，文件之间 Key 范围可能**重叠**，且**无序**，读时需遍历 L0 内多个文件。
- **L1~Ln**：由 Compaction 生成，同层内文件按 Key 范围**有序且不重叠**，读时每层最多查一个文件（二分定位）。

**SST 文件内部结构**：

```
[Data Block 1]
[Data Block 2]
...
[Data Block N]
[Filter Block]            # 如 Bloom Filter
[Metaindex Block]         # 指向 Filter 等元数据
[Index Block]             # 每个 Data Block 的索引（最大 Key + 偏移/大小）
[Footer]                  # 固定长度，记录 Metaindex/Index 的偏移
```

- **Data Block**：实际 Key-Value，按 Key 有序，常用前缀压缩；块大小可配置（如 4KB~64KB）。
- **Index Block**：每个 Data Block 一条索引（如该 Block 最大 Key + 文件偏移与大小），用于二分定位 Data Block。
- **Filter Block**：多为 Bloom Filter，用于快速判断某 Key 是否**不可能**在本文件中，减少无效 IO。
- **Footer**：从文件尾解析，找到 Metaindex 与 Index Block 位置，再解析整颗"索引树"。

**小结**：SST = 多个 Data Block + Index Block + Filter Block + Metaindex + Footer，实现按块读取、按 Key 查找、用 Bloom Filter 过滤。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

### 3.3 MANIFEST（MANIFEST-*）

**作用**：

- 记录数据库**元数据**的变更日志，是恢复"当前视图"的**总账本**。
- 内容在运行过程中**追加**，不原地修改。

**典型内容**：

- 每个时刻的 **Version**：各层（L0~Ln）有哪些 SST 文件。
- 每个 SST 的**元信息**：文件号、列族、层级、Key 范围（smallest/largest）、文件大小等。
- **Compaction**、新增/删除文件等变更记录。
- 序列号、列族配置等元数据。

**恢复逻辑**：

- 打开 DB 时读 `CURRENT` 得到当前 MANIFEST 文件名，再重放 MANIFEST，得到当前所有 SST 的集合与层级，从而恢复出一致的"逻辑视图"。

**滚动**：

- 当 MANIFEST 过大或策略触发时，会做类似 Compaction 的合并，生成新的 MANIFEST 文件，并更新 CURRENT 指向新文件。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

### 3.4 CURRENT

**作用**：

- 一个**小文本文件**，内容为当前生效的 **MANIFEST 文件名**（如 `MANIFEST-000023`）。
- 通过 CURRENT 找到"最新元数据日志"，再读 MANIFEST 恢复完整版本信息。

**使用流程**：

1. 打开 DB 时读 `CURRENT` → 得到当前 MANIFEST 名。
2. 读该 MANIFEST → 恢复 VersionSet（各层 SST 列表等）。
3. 再根据 WAL 做崩溃恢复（如需要）。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

### 3.5 OPTIONS（OPTIONS-* / OPTIONS）

**作用**：

- 将部分 **Options** 持久化到磁盘，便于重启后保持配置或排查历史配置。
- 非必须：不开启则没有 OPTIONS 文件，DB 仍可正常运行。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

### 3.6 LOCK

**作用**：

- **进程级锁**：同一时刻只允许一个进程以写方式打开该 DB 目录，防止多进程同时写导致损坏。
- 打开 DB 时加锁，关闭时释放；若未正常关闭，可能留下 LOCK 文件，需根据运维策略处理（确认无其它进程后再删或等恢复）。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

---

## 四、文件与读写流程的关系

**写入路径**：

```
Put/Delete/Merge
  → WriteBatch
  → 写 WAL（*.log）
  → 写 MemTable
  → MemTable 满 → Immutable MemTable
  → Flush → 生成 L0 SST（*.sst）
  → Compaction → 生成/合并 L1~Ln SST
  → 每次变更记录到 MANIFEST，CURRENT 指向最新 MANIFEST
```

**读取路径**：

```
Get/Iterator
  → 取当前 SuperVersion（基于 MANIFEST 恢复的 Version + MemTable 等）
  → 查 MemTable / Immutable MemTable
  → 查 L0 SST（可能多个，因 L0 无序重叠）
  → 查 L1~Ln SST（每层二分定位到一个 SST，再通过 Index/Filter/Data Block 查）
  → 可选：Block Cache 缓存 Data/Index/Filter Block
```

**崩溃恢复**：

1. 读 **CURRENT** → 当前 **MANIFEST**。
2. 重放 **MANIFEST** → 得到最新 Version（所有 SST 及层级）。
3. 根据 **WAL** 重放未刷盘的写操作 → 重建 MemTable。
4. 必要时将恢复出的 MemTable Flush 成新的 L0 SST，并更新 MANIFEST。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

---

## 五、分析/排查时可关注的点

| 关注点 | 说明 |
|--------|------|
| **WAL 体积与数量** | 过大或过多常表示 Flush/Compaction 跟不上写入，或 `max_total_wal_size` 偏大 |
| **L0 文件数与 write stall** | L0 文件过多会触发 slowdown/stop writes，需看 Compaction 是否及时、参数是否合理 |
| **SST 文件数量与层级** | 各层文件数、单文件大小，反映 Compaction 策略与空间/读放大 |
| **MANIFEST 大小** | 过大时恢复会变慢，RocksDB 会做 MANIFEST 压缩/滚动 |
| **CURRENT 内容** | 确认指向的 MANIFEST 存在且可读，否则无法正确恢复版本 |

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

---

## 六、小结

- **WAL（*.log）**：顺序写日志，保证持久化与崩溃恢复。
- **SST（*.sst）**：分层有序数据文件，内部分块（Data/Index/Filter/Footer），支持高效查找与过滤。
- **MANIFEST + CURRENT**：记录并指向当前"版本视图"（所有 SST 及元数据），是恢复一致性的核心。
- **OPTIONS / LOCK / IDENTITY**：配置持久化、单进程写保护、实例标识等辅助文件。

理解这些文件的角色和格式，有助于分析磁盘占用、恢复流程、以及读写与 Compaction 对文件的影响，便于做容量规划、故障排查和面试作答。

[src: raw/ingested/2技术/rocksdb/rocksdb的文件分析.md]

## Related Pages
- [[RocksDB概述]]
- [[RocksDB LSM-Tree]]
- [[RocksDB写入流程]]
- [[RocksDB读取流程]]
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]
- [[RocksDB性能分析与瓶颈]]

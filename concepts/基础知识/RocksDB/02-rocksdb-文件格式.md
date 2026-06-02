# RocksDB 文件格式

> SST / WAL / MANIFEST / CURRENT / OPTIONS / LOCK 等文件结构详解。

---

## 一、文件类型总览

| 文件类型 | 文件后缀 | 作用 |
|---------|---------|------|
| **SST 文件** | `.sst` | 存储有序的 Key-Value 数据，是 RocksDB 的主要数据文件 |
| **WAL 文件** | `.log` | 预写日志，保证数据持久性和崩溃恢复 |
| **MANIFEST** | `MANIFEST-xxx` | 记录所有 SST 文件的元数据变更历史 |
| **CURRENT** | `CURRENT` | 指向当前最新的 MANIFEST 文件 |
| **OPTIONS** | `OPTIONS-xxx` | 持久化的 RocksDB 配置选项 |
| **LOCK** | `LOCK` | 进程间锁，防止多进程同时打开同一个 DB |
| **IDENTITY** | `IDENTITY` | 存储 DB 的唯一标识 |

### 目录结构示例

```
/data/rocksdb/
├── 000001.sst       # L0 SST 文件
├── 000002.sst       # L1 SST 文件
├── 000003.sst       # L0 SST 文件
├── 000004.log       # WAL 日志
├── 000005.log       # WAL 日志
├── MANIFEST-000006  # 元数据清单
├── CURRENT          # 指向 MANIFEST-000006
├── OPTIONS-000007   # 配置快照
├── LOCK             # 进程锁
└── IDENTITY         # UUID 唯一标识
```

---

## 二、SST 文件详解

### 2.1 整体结构

```
┌──────────────────────────────────────────────┐
│                  SST 文件                      │
├──────────────────────────────────────────────┤
│  [Data Block 1]   Key-Value 数据块           │
│  [Data Block 2]                               │
│  ...                                          │
│  [Data Block N]                               │
├──────────────────────────────────────────────┤
│  [Filter Block]    Bloom Filter 位图          │
├──────────────────────────────────────────────┤
│  [Metaindex Block] 元数据块索引               │
├──────────────────────────────────────────────┤
│  [Index Block]     数据块索引（key→offset）   │
├──────────────────────────────────────────────┤
│  [Footer]          文件尾部，固定长度         │
└──────────────────────────────────────────────┘
```

### 2.2 各块详解

#### Data Block（数据块）

- 存储实际的 Key-Value 对
- 按 Key 有序组织，默认 16KB 一块（`block_size` 可配置）
- 使用**前缀压缩**（Prefix Compression）减小空间占用
- 块内使用**重启点（Restart Point）** 加速查找

**Data Block 内部结构**：
```
┌─────────────────────────────┐
│  Key-Value Entry 1          │
│    - Key Size (varint)      │
│    - Key Data               │
│    - Value Size (varint)    │
│    - Value Data             │
├─────────────────────────────┤
│  Key-Value Entry 2          │
│  ...                        │
├─────────────────────────────┤
│  Restart Points 数组         │
│  （每 N 个 entry 一个重启点）  │
├─────────────────────────────┤
│  Num Restarts (4 bytes)     │
└─────────────────────────────┘
```

#### Index Block（索引块）

- 存储每个 Data Block 的首 Key 和在文件中的偏移量
- 用于加速定位 Data Block，避免遍历全文件
- 结构类似有序数组，通过二分查找定位

**Index Block 结构**：
```
[Block Key 1 → Offset 1]  // Data Block 1 的首 Key 和偏移
[Block Key 2 → Offset 2]  // Data Block 2 的首 Key 和偏移
...
```

#### Filter Block（过滤器块）

- 通常为 Bloom Filter，用于快速判断 Key"不可能"存在
- 按 Data Block 粒度构建，每个 Data Block 对应一个 Bloom 位数组
- 默认 10 bits/key，误判率约 1%

#### Footer（文件尾部）

- 固定长度（通常 48 字节）
- 存储 Metaindex Block 和 Index Block 的偏移量和大小
- 包含魔数（Magic Number）用于文件格式校验

---

## 三、WAL 文件详解

### 3.1 文件格式

```
[Record 1]
  - Record Length: 4 bytes
  - Sequence Number: 8 bytes
  - Record Type: 1 byte (PUT/DELETE/MERGE)
  - Key Length: varint
  - Key Data
  - Value Length: varint (DELETE 无 Value)
  - Value Data
  - CRC32: 4 bytes

[Record 2]
...
[Record N]
```

### 3.2 记录类型

| 类型 | 值 | 说明 |
|------|-----|------|
| kFullType | 0 | 完整记录 |
| kFirstType | 1 | 分段记录的第一段 |
| kMiddleType | 2 | 分段记录的中间段 |
| kLastType | 3 | 分段记录的最后一段 |
| kCompressedBlockType | 4 | 压缩块记录（压缩模式） |

### 3.3 WAL 切换时机

1. WAL 大小超过 `max_total_wal_size`
2. MemTable 切换（Flush 触发）
3. 手动调用 `FlushWAL()`

### 3.4 崩溃恢复流程

```
1. 读取 CURRENT → 获取当前 MANIFEST 文件
2. 加载 MANIFEST → 重建 Version（所有 SST 文件元数据）
3. 找到未 Flush 的 WAL 文件
4. 回放 WAL → 重建 MemTable
5. 将重建的 MemTable 刷盘生成 SST
6. 更新 MANIFEST → 完成恢复
```

---

## 四、MANIFEST 文件详解

### 4.1 作用

MANIFEST 是 RocksDB 的"元数据总账本"，记录数据库状态的所有变更：

- 所有 SST 文件的文件名、所属层级（Level）、Key 范围、文件大小
- 最新的 Version 结构信息（当前数据库快照）
- 历次元数据变更的操作日志（新增/删除文件等）

### 4.2 文件格式

```
[Version Edit 1]
  - Add File: L0, 000001.sst, [key_min, key_max], size=64MB
  - Set Current Version ID: 1

[Version Edit 2]
  - Delete File: L0, 000001.sst
  - Add File: L1, 000003.sst, [key_min, key_max], size=128MB
  - Set Current Version ID: 2

...
```

每次 Version Edit 都是增量追加的，MANIFEST 会不断增长。可通过 `PurgeObsoleteFiles` 机制清理过旧的 MANIFEST。

### 4.3 启动加载流程

```
CURRENT → MANIFEST → 回放所有 Version Edit → 重建最新 Version
```

---

## 五、CURRENT 文件

**作用**：指向当前最新的 MANIFEST 文件。

**文件内容**：
```
MANIFEST-000006
```

只有一行，就是当前 MANIFEST 的文件名。当 MANIFEST 切换时，RocksDB 会原子地更新 CURRENT 文件。

---

## 六、OPTIONS 文件

- 持久化的 RocksDB 配置选项快照
- 用于恢复时确保配置一致
- 文件名格式：`OPTIONS-xxx`（xxx 为文件编号）

---

## 七、LOCK 文件

- 文件锁，防止多进程同时打开同一个 RocksDB DB
- 基于文件系统的 `flock()` 或 `fcntl()` 实现
- 第二个进程尝试打开时会失败

---

## 八、文件生命周期

```
创建 ──→ 使用 ──→ 废弃 ──→ 删除

SST:
  创建：Flush 或 Compaction 完成
  使用：被当前 Version 引用
  废弃：Compaction 合入下层后不再被引用
  删除：PurgeObsoleteFiles 清理

WAL:
  创建：DB Open 或 WAL 切换
  使用：对应 MemTable 未 Flush
  废弃：对应 MemTable 已 Flush
  删除：所有对应 MemTable 都已 Flush

MANIFEST:
  创建：每次 Version Edit 追加
  使用：始终被引用
  废弃：新 MANIFEST 创建后
  删除：PurgeObsoleteFiles 清理旧 MANIFEST
```

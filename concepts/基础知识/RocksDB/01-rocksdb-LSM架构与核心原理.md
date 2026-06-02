# RocksDB LSM 架构与核心原理

> LSM-Tree 架构、核心组件、写入/读取流程、Compaction 策略详解。

---

## 一、LSM-Tree 设计思想

### 1.1 核心思想

- **写入优化**：将随机写转换为顺序写，利用磁盘顺序写入性能优势（顺序 IO 比随机 IO 快 10-100 倍）
- **分层存储**：热数据在内存，冷数据在磁盘，自动分层管理
- **延迟合并**：写入时只追加，Compaction 时合并，分摊写入成本

### 1.2 与 B+ 树对比

| 维度 | B+ 树 | LSM-Tree (RocksDB) |
|-----|------|-------------------|
| **写入方式** | 随机写入（原地更新） | 顺序写入（追加） |
| **写入性能** | 低（随机 IO） | 高（顺序 IO，10-100 倍） |
| **读取性能** | 高（O(log n)） | 中等（多层查询，优化后接近） |
| **空间放大** | 低（无冗余） | 中等（Compaction 前有冗余） |
| **写放大** | 低（1 倍） | 高（5-10 倍，Compaction 重写） |
| **适用场景** | 读多写少（OLTP） | 写多读少（时序、日志） |

---

## 二、核心架构

### 2.1 整体架构

```
┌────────────────────────────────────────────────────────────┐
│                    写入路径                                 │
│  WriteBatch → WAL → MemTable → Immutable → L0 SST         │
└────────────────────────────────────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────────┐
│                    Compaction 路径                          │
│               L0 → L1 → L2 → ... → Ln                      │
└────────────────────────────────────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────────┐
│                    读取路径                                 │
│  MemTable → Immutable → L0 → L1+ → Block Cache            │
└────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

| 组件 | 位置 | 数据结构 | 作用 | 关键参数 |
|-----|------|---------|------|---------|
| **WAL** | 磁盘 | 顺序日志 | 预写日志，保证持久性，崩溃恢复 | `sync_log`, `max_total_wal_size` |
| **MemTable** | 内存 | SkipList（跳表） | 接受写入操作，O(log n) 插入和查询 | `write_buffer_size` (默认 64MB) |
| **Immutable MemTable** | 内存 | SkipList | 只读，等待刷盘，支持并发查询 | `max_write_buffer_number` (默认 2) |
| **L0 SST** | 磁盘 | 无序 SST | Flush 生成，可能重叠，需要遍历 | `level0_file_num_compaction_trigger` (默认 4) |
| **L1-Ln SST** | 磁盘 | 有序 SST | 按 Key 范围有序，不重叠，二分查找 | `max_bytes_for_level_base` (默认 256MB) |
| **Block Cache** | 内存 | LRU Cache | 缓存热数据块，减少磁盘 IO | `block_cache_size` (建议总内存 1/3) |
| **Bloom Filter** | 内存/磁盘 | 位数组 | 快速过滤不存在 Key，减少无效 IO | `bits_per_key` (默认 10) |

---

## 三、写入流程

### 3.1 完整写入路径

```
1. 客户端调用 Put/Delete/Merge
   ↓
2. 封装为 WriteBatch（支持批量操作，原子性保证）
   ↓
3. 写入 WAL（Write-Ahead Log）
   - sync_log=true：强制同步磁盘（性能差，数据安全）
   - sync_log=false：依赖 OS 缓存刷盘（推荐，性能好）
   ↓
4. 写入 MemTable（内存跳表）
   - 支持并发写入（allow_concurrent_memtable_write）
   - Leader-Follower 模型：合并多个 WriteBatch
   ↓
5. MemTable 满（write_buffer_size）
   - 转为 Immutable MemTable
   - 创建新 MemTable
   ↓
6. 后台 Flush 线程
   - Immutable MemTable → L0 SST 文件
   ↓
7. Compaction 触发
   - L0 → L1 → L2 ... → Ln
```

### 3.2 WriteBatch 批量写入

```cpp
WriteBatch batch;
batch.Put("key1", "value1");
batch.Delete("key2");
db->Write(WriteOptions(), &batch);
```

**内部格式**：
- sequence：8 字节序列号
- count：4 字节计数
- data：具体操作记录

### 3.3 WAL 预写日志

**WAL 作用**：
- **持久性保证**：崩溃恢复时重建 MemTable
- **顺序写入**：追加写入，性能高
- **原子性**：一个 WriteBatch 对应一条 WAL 记录

**WAL 文件格式**：
```
[Record Length: 4 bytes]
[Sequence Number: 8 bytes]
[Record Type: 1 byte]   // PUT/DELETE/MERGE
[Key Length: varint]
[Key Data]
[Value Length: varint]   // DELETE 无 Value
[Value Data]
[CRC32: 4 bytes]
```

**WAL 切换时机**：
1. WAL 大小超过 `max_total_wal_size`
2. MemTable 切换（Flush 触发）
3. 手动 Flush（`FlushWAL`）

### 3.4 MemTable 实现细节

**数据结构**：
- **默认实现**：SkipList（跳表）
- **优势**：O(log n) 插入和查询，支持并发写入
- **替代实现**：HashSkipListRep、HashLinkListRep（哈希+跳表）

**生命周期**：
```
创建 → 写入数据 → 写满（write_buffer_size）→ 转为 Immutable → Flush → 销毁
```

**并发写入机制**：
- **无锁跳表**：InlineSkipList 支持并发插入
- **CAS 操作**：Compare-And-Swap 实现无锁同步
- **Group Commit**：多个写入线程合并为 Group

### 3.5 并发写入优化

#### Leader-Follower 模型

```
┌─────────────────────────────────────────────┐
│            写入线程组 (Group)                 │
│  ┌────────┐  ┌────────┐  ┌────────┐        │
│  │Leader  │  │Follower│  │Follower│        │
│  │(WAL)   │  │(等待)  │  │(等待)  │        │
│  └───┬────┘  └───┬────┘  └───┬────┘        │
│      │           │           │             │
│      └───────────┴───────────┘             │
│              ↓                              │
│      并发写入 MemTable                      │
└─────────────────────────────────────────────┘
```

**流程**：
1. 多个写入线程组成 Group
2. 第一个线程成为 Leader，负责写 WAL
3. Leader 完成 WAL 后唤醒 Follower
4. 所有线程并发写入 MemTable

**性能提升**：
- `sync=false`：性能提升约 3 倍
- `sync=true`：性能提升约 2 倍

**流水线写入**（`enable_pipelined_write=true`）：
- 第二个 Group 在第一个 Group 写 MemTable 时，提前开始 WAL 写入
- 进一步提高吞吐量

### 3.6 Write Stall（写入限流）

**触发条件**：
- Immutable MemTable 数量 > `max_write_buffer_number`（默认 2）
- L0 文件数 > `level0_slowdown_writes_trigger`（默认 20）
- Pending Compaction Bytes > `soft/hard_pending_compaction_bytes_limit`

**限流机制**：
- **延迟写入**：写入线程 sleep，降低写入速度
- **阻塞写入**：完全停止写入，等待 Compaction 完成

---

## 四、读取流程

### 4.1 完整读取路径

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

### 4.2 SuperVersion 机制

**作用**：
- **版本快照**：记录当前时刻所有 SST 文件列表
- **引用计数**：防止查询过程中文件被清理
- **原子切换**：Compaction 完成后原子更新 SuperVersion

**组成**：
- MemTable 列表
- Immutable MemTable 列表
- 各级 SST 文件列表（Version）
- Column Family 配置

### 4.3 SST 文件查询优化

**SST 文件结构**：
```
[File Header]
[Data Block 1]
[Data Block 2]
...
[Data Block N]
[Index Block]      ← 数据块索引
[Filter Block]     ← Bloom Filter
[Footer]           ← 文件尾部，包含索引位置
```

**文件定位流程**：
1. **FileMetaData 查询**：通过 min_key 和 max_key 判断 Key 是否在文件范围内
2. **Bloom Filter 过滤**：快速判断 Key 是否"不可能"存在
3. **Index Block 查询**：二分查找定位 Data Block
4. **Data Block 查询**：在 Block 内查找 Key

---

## 五、Compaction 机制

### 5.1 Compaction 作用

1. **合并冗余数据**：同一 Key 的多个版本合并为最新版本
2. **清理删除标记**：Tombstone 在 Compaction 时物理删除
3. **优化读取路径**：减少文件数量，降低读放大
4. **控制空间放大**：及时清理无效数据

### 5.2 Compaction 触发条件

- **L0 触发**：文件数 > `level0_file_num_compaction_trigger`（默认 4）
- **L1+ 触发**：文件大小 > `max_bytes_for_level_base * multiplier`
- **手动触发**：`CompactRange`

### 5.3 Leveled Compaction

**特点**：
- 逐层合并：L0 → L1 → L2 → ... → Ln
- 每层文件按 Key 范围有序，且不重叠
- 空间效率高

**合并过程**：
```
1. 选择 L0 文件（可能多个，有重叠）
2. 选择 L1 文件（与 L0 Key 范围重叠）
3. 多路归并排序
4. 生成新的 L1 文件（有序，不重叠）
5. 删除旧文件
```

**写放大分析**：5-10 倍（取决于层级深度）

**✅ 优势**：读放大低、空间放大低
**❌ 劣势**：写放大高

### 5.4 Universal Compaction

**特点**：
- 全量合并，不区分层级
- 写放大低（2-3 倍）

**触发条件**：
- Size Ratio 触发
- 文件数触发
- 空间放大触发

**适用场景**：写密集型、时序数据、日志系统

**✅ 优势**：写放大低（2-3 倍）
**❌ 劣势**：读放大高、空间放大高

### 5.5 Dynamic Leveled Compaction (7.x+)

- 动态调整每层上限（根据最深层文件大小）
- 空间效率更稳定
- 配置：`level_compaction_dynamic_level_bytes = true`

### 5.6 Compaction 优先级

| 策略 | 说明 |
|------|------|
| **kByCompensatedSize** | 按补偿大小（考虑删除数据） |
| **kOldestLargestSeqFirst** | 优先处理最旧的大文件 |
| **kOldestSmallestSeqFirst** | 优先处理最旧的小文件 |
| **kMinOverlappingRatio** | 最小重叠比例（推荐） |

---

## 六、高级特性

### 6.1 列族（Column Families）

- 将数据划分为多个逻辑组，每个列族独立配置
- 所有列族共享 WAL、Current、Manifest 文件
- 每个列族有独立的 MemTable、SST 文件

**应用场景**：
- 多数据类型分离
- 独立压缩策略
- 资源隔离

### 6.2 事务支持

| 类型 | 说明 |
|------|------|
| **乐观事务** | 无锁，适合冲突少的场景 |
| **悲观事务** | 有锁，适合冲突多的场景 |

### 6.3 快照（Snapshot）

- **一致性读**：快照时刻的数据不变
- **MVCC 支持**：多版本并发控制
- **备份恢复**：基于快照备份

### 6.4 Env 插件机制

Env 是 RocksDB 对操作系统环境的抽象接口：

**核心功能**：
- 文件系统操作（创建/删除/读写）
- 线程管理
- 时间获取
- 同步原语

**典型实现**：
- **BlueRocksEnv**：Ceph BlueStore 直接管理裸设备
- **EnvLibrados**：对接 Ceph RADOS 对象存储
- **SPDK Env**：NVMe 用户态驱动

**EnvWrapper** 设计模式：用户只需覆盖感兴趣的方法，其余默认转发。

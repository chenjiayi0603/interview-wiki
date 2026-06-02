# RocksDB 面试考点速查

> 面试高频 Q&A、KeyDB 存算分离项目、三大放大、Compaction 策略对比。

---

## 一、高频面试 Q&A

### Q1: RocksDB 的写入流程是怎样的？

```
WriteBatch → WAL → MemTable → Immutable MemTable → L0 SST → Compaction → Ln
```

1. 客户端调用 Put/Delete/Merge，封装为 WriteBatch
2. 写入 WAL（预写日志，保证持久性）
3. 写入 MemTable（内存跳表，O(log n)）
4. MemTable 写满后转为 Immutable MemTable，创建新 MemTable
5. 后台 Flush 将 Immutable MemTable 刷盘为 L0 SST
6. Compaction 合并 L0→L1→...→Ln

### Q2: RocksDB 的读取流程是怎样的？

```
MemTable → Immutable MemTable → L0 SST → L1+ SST → Block Cache → 磁盘
```

1. 获取 SuperVersion（数据版本快照）
2. 查 MemTable（跳表）
3. 查 Immutable MemTable
4. 遍历 L0 SST（可能重叠）
5. 二分查找 L1+ SST（每层一个文件）
6. Bloom Filter 预过滤，Index Block 定位 Data Block
7. Block Cache 命中则直接返回，否则磁盘读取

### Q3: LSM-Tree 相比 B+ 树的优缺点？

| 维度 | LSM-Tree | B+ 树 |
|-----|---------|-------|
| **写入性能** | ✅ 高（顺序写） | ❌ 低（随机写） |
| **读取性能** | ❌ 中等（多层查询） | ✅ 高（单次查询） |
| **写放大** | ❌ 5-10 倍 | ✅ 1 倍 |
| **空间放大** | ❌ 中等 | ✅ 低 |
| **适用场景** | 写多读少（时序、日志） | 读多写少（OLTP） |

### Q4: 什么是写放大？如何优化？

**定义**：物理写入数据量 / 逻辑写入数据量。RocksDB 中主要由 Compaction 引起。

**优化手段**：
- 使用 Universal Compaction（写放大 2-3x）
- 增大 MemTable（减少 Flush 频率）
- 调整 Compaction 触发阈值
- 启用子 Compaction（缩短单次时间）

### Q5: 什么是 Write Stall？如何避免？

**原因**：
- Immutable MemTable 超过上限
- L0 文件过多
- Pending Compaction Bytes 超标

**避免方法**：
- 增加后台线程（`max_background_jobs`）
- 增大 MemTable 数量和大小
- 调整 Compaction 策略
- 使用 Rate Limiter

### Q6: Leveled Compaction 和 Universal Compaction 的区别？

| 维度 | Leveled | Universal |
|-----|---------|-----------|
| **结构** | 严格层级（L0~Ln） | 无层级，文件并列 |
| **写放大** | 高（5-10x） | 低（2-3x） |
| **读放大** | 低 | 高 |
| **空间放大** | 低 | 高 |
| **适用** | 读多写少、通用场景 | 写密集型、时序数据 |

### Q7: Bloom Filter 在 RocksDB 中的作用？

**作用**：快速判断 Key"不可能"存在于当前 SST 文件，减少无效磁盘 IO。

**原理**：
```
Key → k 次哈希 → 映射到 Bloom 位数组
查询时：
  - 任一哈希位为 0 → Key 一定不存在 → 跳过该 SST
  - 所有哈希位为 1 → Key 可能存在 → 继续查询
```

**配置**：`bits_per_key=10`（默认），误判率约 1%。

### Q8: Block Cache 缓存哪些内容？如何配置？

**缓存内容**：
- Data Block（实际 Key-Value 数据）
- Index Block（数据块索引）
- Filter Block（Bloom Filter）

**配置建议**：
```cpp
table_options.block_cache = NewLRUCache(total_memory / 3);
table_options.cache_index_and_filter_blocks = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;
```

### Q9: WAL 的作用和配置？

**作用**：
- 保证持久性（崩溃恢复时重建 MemTable）
- 顺序写入，性能高

**配置**：
```cpp
options.sync_log = false;  // 推荐：异步刷盘，性能好
// 或
options.sync_log = true;   // 强安全，性能差
```

**崩溃恢复流程**：
```
CURRENT → MANIFEST → Version → WAL 回放 → 重建 MemTable → Flush 到 SST
```

### Q10: SuperVersion 是什么？

**定义**：RocksDB 当前数据版本的快照，包含：
- MemTable 列表
- Immutable MemTable 列表
- 各级 SST 文件列表（Version）
- Column Family 配置

**作用**：
- 提供一致性读视图
- 引用计数防止文件在查询中被清理
- Compaction 完成后原子切换

### Q11: RocksDB 的 Env 插件机制？

Env 是 RocksDB 对操作系统的抽象接口：
- 文件系统操作
- 线程管理
- 时间获取
- 同步原语

**典型实现**：
- BlueRocksEnv：Ceph BlueStore 直接管理裸设备
- EnvLibrados：对接 Ceph RADOS
- SPDK Env：NVMe 用户态驱动

### Q12: RocksDB 7.x 相比 6.x 有哪些重要改进？

| 改进 | 说明 |
|------|------|
| **Compaction 阻塞修复** | 修复 Pending Bytes 计算放大，写入 QPS 波动 < 5% |
| **分区索引** | 多级索引，顶层常驻内存，减少 Swap 竞争 |
| **Zstd 压缩** | 新增 Zstd 算法，高压缩率+速度 |
| **增量 Compaction** | 分批次处理，降低单次资源占用 |
| **jemalloc 集成** | 减少内存碎片 |
| **动态层级** | 空间效率更稳定 |

### Q13: 列族（Column Family）的作用？

- 将数据划分为多个逻辑组，独立配置
- 共享 WAL/Manifest/Current
- 独立 MemTable/SST/Compaction 策略

**应用场景**：
- 多数据类型分离（时序数据与元数据）
- 独立压缩策略
- 资源隔离

### Q14: RocksDB 的磁盘带宽限制和优化？

**磁盘带宽限制**：
- 磁盘 IOPS 不足（写放大 5-10 倍）
- 与前台请求争抢

**优化策略**：
- 使用 Rate Limiter 限制 Compaction IO
- 启用 Direct IO 绕过 Page Cache
- 使用压缩减少 IO 量
- 调大 `bytes_per_sync` 减少 sync 次数

### Q15: RocksDB 和 Redis 的适用场景区别？

| 维度 | RocksDB | Redis |
|-----|---------|-------|
| **数据类型** | 嵌入式 KV 存储引擎 | 缓存/数据库服务 |
| **数据持久性** | 磁盘持久化 | 内存为主 + 持久化 |
| **写入性能** | 高（顺序写） | 极高（纯内存） |
| **读取性能** | 中等（多层查询） | 极高（纯内存） |
| **数据量** | 可超过内存（TB 级） | 受内存限制 |
| **典型场景** | 数据库引擎、流计算状态 | 缓存、会话、排行榜 |

---

## 二、KeyDB 存算分离项目

### 项目背景

基于 KeyDB + RocksDB 实现存算分离架构，满足大容量 KV 存储场景（TB 级），解决纯内存 Redis 成本高、容量受限的问题。

### 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                     KeyDB 存算分离架构                        │
├─────────────────────────────────────────────────────────────┤
│  计算层：KeyDB (多线程 Redis 兼容)                           │
│  - 多线程事件循环，突破单线程瓶颈                            │
│  - 兼容 Redis 协议，业务无感迁移                             │
│  - 热点数据本地缓存 (Caffeine/Guava)                        │
│                                                             │
│  存储层：RocksDB + OBS 对象存储                              │
│  - RocksDB：本地 SSD 作为缓存层                             │
│  - OBS：海量数据持久化，成本降低 70%                        │
│  - RocksDB Env 插件对接 OBS 存储后端                        │
└─────────────────────────────────────────────────────────────┘
```

### 核心技术要点

#### 1. RocksDB VFS 抽象层

```cpp
// 自定义 Env 对接 OBS 对象存储
class OBSEnv : public rocksdb::EnvWrapper {
public:
    Status NewSequentialFile(const std::string& fname,
                             std::unique_ptr<SequentialFile>* result,
                             const EnvOptions& options) override {
        // 本地文件 → 本地 SSD
        // 冷数据文件 → OBS 远端存储
    }
    // ...
};
```

#### 2. 冷热分层策略

```
写入路径：
  KeyDB → RocksDB WAL(本地 SSD) → MemTable → Flush → L0 SST(本地 SSD)
  
冷热迁移：
  热数据：本地 SSD（Block Cache + L0/L1 SST）
  冷数据：异步迁移到 OBS 对象存储
  
读取路径：
  KeyDB 请求 → 本地 SSD 查询 → 命中则返回
                → 未命中则从 OBS 拉取 → 缓存到本地 SSD
```

#### 3. gRPC 元数据服务

- 管理数据分片和节点状态
- 版本号/epoch 机制保证数据一致性
- 故障转移自动触发

### 面试话术（STAR 框架）

**S (Situation)**：
业务需要 TB 级 KV 存储，纯内存 Redis 成本高（单 GB 约 100 元/月），且容量受限无法满足海量数据存储。

**T (Task)**：
设计存算分离架构，实现 Redis 兼容协议，将冷数据下沉到对象存储，降低 70% 存储成本，同时保证 10 万+ QPS。

**A (Action)**：
1. 选用 KeyDB（多线程 Redis 兼容）作为计算层，突破单线程性能瓶颈
2. RocksDB 作为存储引擎，通过 Env 插件对接 OBS 对象存储
3. 冷热分层：本地 SSD 缓存热数据，OBS 存储冷数据
4. gRPC 元数据服务 + 版本号/epoch 机制保证一致性
5. 兼容 Redis 协议，客户端无感迁移

**R (Result)**：
- 存储成本降低约 30%（vs 全内存方案）
- 支持 10 万+ QPS 读写
- 容量扩展到 TB 级
- 99.99% 可用性

### 项目中遇到的挑战

| 挑战 | 解决方案 |
|------|----------|
| OBS 网络延迟 | 本地 SSD 缓存热数据，Block Cache 加速 |
| RocksDB 写放大 | 开启 Universal Compaction，增大 MemTable |
| 数据一致性 | gRPC 元数据 + 版本号/epoch 机制 |
| 兼容 Redis 协议 | 使用 KeyDB 多线程架构 |

---

## 三、Redis 存储型方案性能对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **Redis 纯内存** | 极高性能（微秒级） | 成本高、容量受限 | 热数据缓存 |
| **KeyDB + RocksDB** | 大容量、低成本 | 读写放大、延迟增加 | TB 级 KV 存储 |
| **Pika** | Redis 兼容、持久化 | 功能不完整 | 大容量缓存 |
| **Kvrocks** | Redis 兼容、RocksDB 后端 | 生态较新 | 新项目选型 |
| **SSDB** | 轻量级、LevelDB 后端 | 性能一般 | 简单 KV 场景 |

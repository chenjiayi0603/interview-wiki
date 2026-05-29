# RocksDB 与 KeyDB 串行化对比

## 三、RocksDB 与 KeyDB 串行化对比详解

RocksDB 的串行化点远多于 KeyDB（WriteThread、mutex_、WAL、Sequence、Compaction 等），而 KeyDB 主要是 `g_lock` 一把全局锁。本节从定位、数据模型、持久化需求等维度分析差异根因。

### 3.1 定位与数据模型对比

| 维度 | RocksDB | KeyDB |
|------|---------|-------|
| **定位** | 嵌入式存储引擎（库） | 内存数据库服务端 |
| **数据主存** | 磁盘（SST）+ 内存（MemTable） | 内存（dict）为主 |
| **持久化** | WAL + LSM 落盘，强一致性 | AOF/RDB 可选，异步刷盘 |
| **使用方式** | 被 MySQL、TiKV、Cassandra 等嵌入 | 独立进程，客户端连接 |
| **典型场景** | 分布式存储底层、大容量持久化 | 缓存、会话、实时数据 |

**RocksDB 定位**：作为**嵌入式存储引擎**，以库的形式被上层系统链接，不直接对外提供网络服务。设计目标是在保证持久化、崩溃恢复、多版本读等语义的前提下，尽可能提高吞吐。

**KeyDB 定位**：作为**独立内存数据库服务**，直接接受客户端连接。数据主要在内存，持久化是可选增强，设计目标是低延迟、高 QPS。

### 3.2 RocksDB 串行化多的根本原因

#### 3.2.1 持久化与崩溃恢复

- **WAL 顺序写**：WAL 是追加式单文件，多线程并发写会破坏记录边界，崩溃恢复无法正确解析。必须串行写入。
- **fsync 串行 I/O**：`sync=true` 时每次写入需 fsync，单次延迟 50~200μs（NVMe），本质是串行磁盘操作。
- **Sequence 全局递增**：恢复时按 Sequence 重放 WAL，必须保证全序，分配必须在 mutex 保护下完成。

#### 3.2.2 LSM 结构带来的状态管理

- **Version / Manifest**：LSM 各层 SST 文件列表、元数据由 VersionSet 管理，Flush/Compaction 会修改，需 mutex 保护。
- **Compaction 调度**：任务队列、文件选择、进度跟踪等共享状态需同步。
- **SST 元数据**：文件创建、删除、引用计数等操作需原子性。

#### 3.2.3 Snapshot 与多版本读

- **Sequence 全序**：`GetSnapshot()` 返回的 Sequence 用于一致性读，必须与写入顺序一致。
- **分配在锁内**：Sequence 分配、LastSequence 更新必须在 `mutex_` 保护下，否则多版本可见性会错乱。

#### 3.2.4 嵌入式多线程使用方式

- **同一进程多线程**：RocksDB 被嵌入后，业务线程、后台线程、Compaction 线程等可能同时访问，需细粒度同步。
- **无独立调度层**：不像 KeyDB 有独立 Worker 池，RocksDB 的“用户线程”即调用方线程，并发模式更复杂。

### 3.3 KeyDB 串行化少的原因

| 因素 | 说明 |
|------|------|
| **数据在内存** | dict 读写为主，无磁盘 I/O 串行约束 |
| **临界区短** | 单条命令：dict 查找/插入 + 可选 AOF 追加，约 100~800ns |
| **一把 g_lock** | 命令执行前后加锁即可，无需 WAL/Compaction/Version 等复杂状态 |
| **持久化异步** | AOF 多数为 `appendfsync everysec`，fsync 在 BIO 线程，不阻塞命令执行 |
| **MVCC 实现不同** | 有 MVCC，但版本分配不依赖全局串行，见下文 |

**KeyDB 有 MVCC，为何串行化不受影响？**

KeyDB 支持 MVCC（多版本并发控制），但版本管理方式与 RocksDB 不同，**不会引入额外的全局串行化**：

| 对比项 | RocksDB | KeyDB |
|--------|---------|-------|
| **版本粒度** | 全局 Sequence，全库唯一递增 | 按 key 的版本链（MVCCValue），或快照 epoch |
| **分配时机** | 每次写都在 mutex 内分配，与 WAL/MemTable 强绑定 | 在 g_lock 临界区内分配，或 CAS 按 key 分配 |
| **可见性依赖** | Snapshot 依赖全局 Sequence，Compaction 依赖 Sequence 边界 | 快照依赖 epoch/时间戳，无 LSM Compaction |
| **额外串行化** | Sequence 需独立 mutex 保护，与 WAL、Version 等协同 | g_lock 已串行化写，版本分配在锁内完成，无额外锁 |

**核心差异**：RocksDB 的 Sequence 是 **WAL、MemTable、SST、Compaction 四者共用的唯一版本号**：

- **WAL**：每条 Record 带 Sequence，崩溃恢复按 Sequence 顺序重放
- **MemTable**：每个 key 带 Sequence，读时用 Snapshot 的 Sequence 判断可见性
- **SST**：Flush/Compaction 落盘时保留 Sequence，读/Compaction 时用于可见性与回收
- **Compaction**：根据「最老 Snapshot 的 Sequence」决定可删除的旧版本

因此必须在持 mutex 的整条写路径中原子分配，否则四者无法一致。KeyDB 的 MVCC 版本可**按 key 独立**（CAS 更新 mvcc_head），或由 g_lock 内的写操作隐式赋予，不要求这种全局共用版本号，故不增加串行化点。

### 3.4 串行化点对比小结

**串行化点流程图（★ 为加锁/串行处）：**

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  RocksDB 写入路径（串行化/加锁点）                                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  用户线程 Put/Write                                                                      │
│       │                                                                                  │
│       ▼                                                                                  │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ JoinBatchGroup                      │  ← CAS 竞争 newest_writer_（串行化）          │
│  │   LinkOne: 链入队列                   │  ← Follower: AwaitState 阻塞（mutex+cond）   │
│  └─────────────────┬─────────────────────┘                                               │
│                    │ Leader 进入                                                         │
│                    ▼                                                                     │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ mutex_.Lock()                       │  ← DB 全局锁                                 │
│  │   PreprocessWrite / EnterAsBatchGroupLeader                                           │
│  └─────────────────┬─────────────────────┘                                               │
│                    │                                                                     │
│                    ▼                                                                     │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ WriteToWAL                         │  ← 顺序写，sync 时 fsync 串行 I/O             │
│  │   Sequence 分配（在 mutex 内）       │  ← Sequence 原子分配（串行）                   │
│  └─────────────────┬─────────────────────┘                                               │
│                    │                                                                     │
│                    ▼                                                                     │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ InsertInto MemTable                │  ← 默认 Leader 串行；并发模式 SkipList CAS   │
│  │   SetLastSequence                    │  ← 在 mutex 内                                │
│  └─────────────────┬─────────────────────┘                                               │
│                    │                                                                     │
│                    ▼                                                                     │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ mutex_.Unlock()                    │                                               │
│  │   ExitAsBatchGroupLeader             │  ← SetState: Writer::StateMutex + notify_one   │
│  └─────────────────────────────────────┘  ← Follower 唤醒（加锁更新状态）              │
│                                                                                         │
│  后台（Flush/Compaction 线程池）：                                                       │
│    - 与前台共用同一 mutex_（DBImpl::mutex_）                                              │
│    - Flush/Compaction 完成时需 mutex_ 安装新 Version、更新 Manifest                       │
│    - 任务调度另有队列锁（与 mutex_ 不同）                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────┘

**DBImpl::mutex_ 使用场景（简约）**：

| 场景 | 说明 |
|------|------|
| 前台写路径 | PreprocessWrite、Sequence 分配、WAL/MemTable 写前检查、SetLastSequence |
| 后台 Flush | 完成时安装新 Version、更新 Manifest、更新 flush_queue_ |
| 后台 Compaction | 完成时安装新 Version、更新 compaction_queue_、写 Manifest |
| 其他 | Snapshot 创建/释放、CF 元数据变更（SuperVersion 切换、SST 文件列表） |

保护对象及用途：

| 成员 | 用途 |
|------|------|
| `versions_` | LSM 元数据：各层 SST 文件列表、Sequence 上界、SuperVersion |
| `alive_log_files_` | 当前活跃 WAL 文件列表，用于 WAL 回收与恢复 |
| `flush_queue_` | 待 Flush 的 CF 队列，调度 MemTable → SST |
| `compaction_queue_` | 待 Compaction 的 CF 队列，调度层间合并 |
| `flush_jobs_` | 正在执行的 Flush 任务，用于去重与状态跟踪 |
| `snapshots_` | 活跃 Snapshot 列表，决定可见性与可回收的 Sequence 范围 |

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  KeyDB 命令执行路径（串行化/加锁点）                                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  Worker 线程                                                                             │
│       │                                                                                  │
│       ▼  读取命令、解析 RESP（无锁）                                                      │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ aeAcquireLock() → g_lock.lock()   │  ← 全局锁，命令执行前后                         │
│  └─────────────────┬─────────────────────┘                                               │
│                    │                                                                     │
│                    ▼                                                                     │
│  ┌─────────────────────────────────────┐                                               │
│  │   call() 执行命令                    │  ← 持 g_lock：读写 redisDb、过期字典、Pub/Sub  │
│  │   feedAppendOnlyFile / 复制          │  ← 临界区约 100~800ns                          │
│  └─────────────────┬─────────────────────┘                                               │
│                    │                                                                     │
│                    ▼                                                                     │
│  ┌─────────────────────────────────────┐                                               │
│  │ ★ aeReleaseLock() → g_lock.unlock() │                                               │
│  └─────────────────────────────────────┘                                               │
│       │                                                                                  │
│       ▼  写响应（无锁）                                                                   │
│                                                                                         │
│  后台：BIO 任务队列 ← bio_mutex（与主路径分离）                                           │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**结论**：RocksDB 串行化点多且细，源于 LSM 持久化、WAL、Sequence、Version 等；KeyDB 以内存为主，单 g_lock 即可，串行化点少。

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-三、RocksDB-与-KeyDB-串行化对比详解.md]
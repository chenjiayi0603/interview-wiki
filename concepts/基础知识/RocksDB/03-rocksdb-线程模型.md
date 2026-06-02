# RocksDB 线程模型

> 线程架构、Leader-Follower 模型、并发控制、序列化点分析。

---

## 一、线程架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户写入线程                                  │
│    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐               │
│    │写线程1│  │写线程2│  │写线程3│  │写线程4│  │写线程5│               │
│    └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘               │
│       └─────────┴─────────┴─────────┴─────────┴──┘                │
│                        │ Group Commit                              │
│                        ▼                                           │
│             ┌──────────────────┐                                   │
│             │  WriteThread     │  ←  Leader-Follower 模型           │
│             │  串行化 WAL 写入   │                                   │
│             └────────┬─────────┘                                   │
│                      │                                             │
│           ┌──────────▼──────────┐                                  │
│           │  MemTable 并发写入    │  ← allow_concurrent_memtable_write│
│           └──────────┬──────────┘                                  │
│                      │                                             │
├──────────────────────┼─────────────────────────────────────────────┤
│                      ▼                                             │
│           ┌──────────────────┐   max_background_flushes            │
│           │  Flush 线程池     │  ← 后台线程池                       │
│           │  (后台线程)       │                                     │
│           └────────┬─────────┘                                     │
│                    │                                               │
│                    ▼                                               │
│           ┌──────────────────┐   max_background_compactions        │
│           │  Compaction 线程池 │  ← 后台线程池                      │
│           │  (后台线程)       │                                     │
│           └────────┬─────────┘                                     │
│                    │                                               │
│                    ▼                                               │
│              磁盘 IO (SST 写入/读取)                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、Leader-Follower 写入模型

### 2.1 模型概述

RocksDB 通过 Leader-Follower 模型实现高效的并发写入：

```
多个写入线程 → JoinBatchGroup → 产生一个 Leader
                                → 其余为 Follower

Leader 职责：
  1. 收集 Group 中所有 WriteBatch
  2. 合并写入 WAL（串行）
  3. 设置并发写入状态
  4. 唤醒所有 Follower

Follower 职责：
  1. 在 JoinBatchGroup 中等待
  2. 被唤醒后并发写入 MemTable
```

### 2.2 完整流程

```
1. 初始写入阶段
   客户端调用 Put/Merge/Delete
   操作封装到 WriteBatch
   调用 DBImpl::Write 方法

2. JoinBatchGroup 阶段
   调用 WriteThread::LinkOne 将请求挂到链表
   判断是否为链表第一个请求：
     ✅ 是：成为 Leader，负责写 WAL
     ❌ 否：成为 Follower，调用 AwaitState 等待

3. Leader 处理流程
   ┌─ PreprocessWrite：
   │   检查 WAL 大小，必要时 SwitchWAL
   │   检查 MemTable 大小，必要时 HandleWriteBufferFull
   │   检查是否需要 DelayWrite（写入限流）
   │
   ├─ EnterAsBatchGroupLeader：
   │   选择可组合的 batch 构建 Write Group
   │   使用双向链表组织 Write Group
   │
   ├─ WriteToWAL：
   │   合并 group 中所有 batch 的 rep_
   │   调用 log_writer->AddRecord 写入 WAL
   │   根据 manual_flush_ 决定是否立即刷盘
   │
   └─ LaunchParallelMemTableWriters：
       设置并发写入状态
       Leader 和 Follower 并发写入各自 MemTable

4. Follower 处理流程
   在 JoinBatchGroup 中等待
   被 Leader 设为 STATE_PARALLEL_MEMTABLE_WRITER 后唤醒
   并发写入 MemTable
```

### 2.3 流水线写入

通过 `enable_pipelined_write=true` 启用：

```
时间 →
┌───────────────────┐
│ Group 1 WAL 写入   │ ← 串行
└───────────────────┘
                    ┌───────────────────┐
                    │ Group 1 MemTable  │ ← 并发写入
                    │ Group 2 WAL 写入   │ ← 与 Group 1 MemTable 流水线重叠
                    └───────────────────┘
                                         ┌───────────────────┐
                                         │ Group 2 MemTable  │
                                         │ Group 3 WAL 写入   │
                                         └───────────────────┘
```

---

## 三、后台线程池

### 3.1 线程池类型

| 线程池 | 配置参数 | 默认值 | 作用 |
|--------|---------|--------|------|
| **Flush 线程池** | `max_background_flushes` | 1 | 将 Immutable MemTable 刷盘为 L0 SST |
| **Compaction 线程池** | `max_background_compactions` | 1 | 执行 L0→Ln 的 Compaction |
| **高优先级线程池** | `max_background_jobs` | 2 | 统一管理 Flush + Compaction |

**推荐配置**（7.x+）：
```cpp
options.max_background_jobs = 6;  // 统一管理所有后台任务
// 自动分配 Flush 和 Compaction 线程数
```

### 3.2 后台任务调度

```
RocksDB 使用线程池（ThreadPool）管理后台任务：

高优先级池 HighPriPool：
  - Flush 任务（Immutable MemTable → L0 SST）
  - 优先执行，避免 Write Stall

低优先级池 LowPriPool：
  - Compaction 任务（L0 → L1 → ... → Ln）
  - 后台异步执行

线程池内部使用：
  - 有界队列（bounded queue）管理任务
  - 支持动态增减线程
  - 支持任务优先级
```

### 3.3 Flush 流程

```
Immutable MemTable → 放入 Flush 任务队列
                ↓
HighPriPool 线程取任务
                ↓
创建 L0 SST 文件
  1. 遍历 MemTable 写入 Data Block
  2. 构建 Index Block 和 Filter Block
  3. 写入 Footer
  4. 更新 MANIFEST
                ↓
删除 Immutable MemTable
                ↓
通知 Compaction 作业
```

### 3.4 Compaction 流程

```
触发条件满足 → 放入 Compaction 任务队列
                ↓
LowPriPool 线程取任务
                ↓
执行 Compaction
  1. 选择输入文件（Source + Target）
  2. 多路归并排序
  3. 生成新的 SST 文件
  4. 删除旧文件
  5. 更新 MANIFEST
  6. 原子切换 SuperVersion
```

---

## 四、序列化点分析

### 4.1 WriteThread 序列化

```
WriteThread::LinkOne → 链表操作 → CAS 原子操作
WriteThread::WriteToWAL → 串行写入 → 一个 Group 只有一个 Leader 写 WAL

关键锁：
  - mutex_：保护 WAL 写入
  - WriteThread 内部状态机：STATE_* 切换
```

### 4.2 全局序列化点

| 序列化点 | 锁/机制 | 影响 |
|---------|---------|------|
| **WAL 写入** | `log_writer_mutex` | 一个 Group 内串行 |
| **MemTable 切换** | `mutex_` | 创建新 MemTable 时上锁 |
| **Version 更新** | `mutex_` | Compaction 完成更新 Manifest |
| **SuperVersion 切换** | 原子指针 | 无锁（RCU 风格） |
| **Flush 调度** | `mutex_` | Flush 触发检查 |
| **Compaction 调度** | `mutex_` | Compaction 触发检查 |

### 4.3 与 KeyDB 的序列化对比

```
RocksDB 的序列化点很多：
  - WriteThread（Leader 写 WAL）
  - mutex_（全局元数据锁）
  - WAL 写入（IO 序列化）
  - Sequence 分配（原子递增）
  - Compaction 调度

KeyDB 的序列化点：
  - g_lock（全局事件循环锁）
  - 单线程处理所有命令
```

---

## 五、并发配置参数

### 5.1 核心配置

```cpp
// 写入并发
options.allow_concurrent_memtable_write = true;   // 并发写 MemTable
options.enable_pipelined_write = true;             // 流水线写入

// 后台线程
options.max_background_jobs = 6;                   // 后台任务总数
// 或单独配置：
// options.max_background_flushes = 2;
// options.max_background_compactions = 4;

// 并行 Compaction
options.max_subcompactions = 4;                    // 子 Compaction 并行数
```

### 5.2 配置建议

| 场景 | max_background_jobs | max_subcompactions | 说明 |
|------|--------------------|-------------------|------|
| HDD | 2-4 | 1 | 磁盘 IO 是瓶颈，不宜过多线程 |
| SATA SSD | 4-6 | 2-4 | SSD 支持并发 IO |
| NVMe SSD | 6-12 | 4-8 | 高 IOPS，可加大并发 |
| CPU 瓶颈 | 2-4 | 1-2 | 减少 Compaction 线程数 |
| 内存受限 | 2-4 | 1 | 减少 Compaction 内存占用 |

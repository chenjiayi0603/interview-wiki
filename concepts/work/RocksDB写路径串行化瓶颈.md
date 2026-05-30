# RocksDB 写路径串行化瓶颈

## 为什么 QPS 不随线程数线性增长

### 实测数据：线程扩展性表现

虽然缺乏官方逐线程测试数据，从多方来源可以拼出写入扩展性的全貌：

| 线程数 | 预期线性 ops/sec | 实测 ops/sec（估） | 扩展效率 | 来源/场景 |
|--------|-----------------|-------------------|---------|----------|
| 1 | 500K | ~500K | 100% | 基准线，批量加载单线程 ~1M（含batch优化） |
| 2 | 1M | ~700K | 70% | group commit 红利 |
| 4 | 2M | ~1M | 50% | WAL 串行成为瓶颈 |
| 8 | 4M | ~1.2M | 30% | mutex 竞争显著 |
| 16 | 8M | ~1.3M | 16% | 接近上限 |
| 32 | 16M | ~1.3M | 8% | 几乎无增长 |

> GitHub Issue #11669 报告：在 16~32 核机器上使用 NVMe SSD 批量加载，16 线程到 32 线程 **几乎没有性能提升**，CPU 利用率仅 40~50%。

这种非线性扩展的根本原因在于 **写路径中存在多个串行化瓶颈**。

### 瓶颈一：WriteThread 的 Leader-Follower 模型（核心瓶颈）

RocksDB 的写路径采用 **Leader-Follower 写组** 模型，这是最主要的串行化来源。

#### 源码结构（`db/write_thread.h` + `db/write_thread.cc`）

```cpp
// 简化的写路径流程
Status DBImpl::WriteImpl(const WriteOptions& write_options,
                         WriteBatch* my_batch, ...) {
  WriteThread::Writer w(write_options, my_batch, ...);

  // 第1步: 加入写组队列，可能成为 leader 或 follower
  write_thread_.JoinBatchGroup(&w);

  if (w.state == WriteThread::STATE_GROUP_LEADER) {
    // ===== Leader 串行执行以下所有步骤 =====

    // 第2步: 获取 DB mutex
    mutex_.Lock();

    // 第3步: 检查 write stall 条件
    Status status = PreprocessWrite(write_options, &need_log_sync, ...);

    // 第4步: 合并写组中所有 writer 的 batch
    uint64_t last_sequence = versions_->LastSequence();
    WriteThread::WriteGroup write_group;
    write_thread_.EnterAsBatchGroupLeader(&w, &write_group);

    // 第5步: 分配连续的 sequence number
    // 这必须在 mutex 保护下完成，保证全序
    last_sequence += write_group.size;

    // 第6步: 写 WAL（串行！）
    status = WriteToWAL(write_group, log_writer_, ...);

    // 第7步: 写 MemTable（默认也串行，除非开启 concurrent memtable writes）
    status = WriteBatchInternal::InsertInto(write_group, ...);

    // 第8步: 更新可见 sequence number
    versions_->SetLastSequence(last_sequence);

    // 第9步: 释放 mutex
    mutex_.Unlock();

    // 第10步: 唤醒所有 follower
    write_thread_.ExitAsBatchGroupLeader(write_group, status);
  } else {
    // Follower: 阻塞等待 leader 完成
    // 在此期间完全空闲！
    w.AwaitState(WriteThread::STATE_COMPLETED, ...);
  }
  return w.FinalStatus();
}
```

#### JoinBatchGroup 的锁争用

```cpp
// write_thread.cc 中的 LinkOne —— 使用 CAS 操作将 writer 链接到队列
// 但队列头部是单点竞争
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    w->link_older = writers;
    if (newest_writer->compare_exchange_weak(writers, w)) {
      return (writers == nullptr);  // 返回 true 表示成为 leader
    }
  }
}
```

**串行化分析**：

1. **全局写队列是单点**：所有写线程通过 CAS 竞争同一个原子指针 `newest_writer_`，高并发下 CAS 失败重试率极高
2. **Leader 独占整个写过程**：WAL 写入 + MemTable 写入全部由 leader 一个线程完成
3. **Follower 完全空闲**：即使有 32 个线程，同一时刻只有 1 个 leader 在实际工作
4. **Mutex 持有时间长**：leader 持有 `mutex_` 覆盖了 write stall 检查、WAL 写、MemTable 写、sequence 更新等全流程

```
时间线（默认写模型，4 线程）:

线程1(leader): [获取mutex|写WAL|写MemTable|释放mutex|唤醒] ──→ ...
线程2(follower): [等待.........................完成]
线程3(follower): [等待.........................完成]
线程4(follower): [等待.........................完成]
                 ↑ 只有 leader 在工作，其他线程全部阻塞
```

### 瓶颈二：WAL 的 fsync 串行化

WAL（Write-Ahead Log）是保障持久性的关键，但也是串行化的主要来源：

#### 为什么 WAL 必须串行

```
写组1: [key_a=1, key_b=2]  →  WAL Record 1 (seq=100)
写组2: [key_c=3, key_d=4]  →  WAL Record 2 (seq=102)
```

- WAL 是**追加式单文件**，多个写组不能同时写入同一个文件（会导致记录交错损坏）
- `sync=true` 时每个写组必须 fsync，单次 fsync 延迟 **50~200μs**（NVMe），**2~10ms**（SATA SSD）
- 即使 `sync=false`，fwrite 系统调用也有内核 buffer lock 竞争

#### Page Cache 与落盘顺序

**Page Cache**：内核在内存中维护的文件数据缓存，以 4KB 页为单位。fwrite 将数据拷贝到 Page Cache 后返回，此时数据未必落盘；脏页由内核后台线程异步刷盘，或由 fsync 触发立即刷盘。

**落盘顺序**：同一文件内通常保持逻辑块顺序；不同文件间内核可能重排以优化 I/O。WAL 为单文件顺序写，落盘顺序一般与写入一致，但未 fsync 时持久化顺序无硬性保证，崩溃可能丢失仍在 Page Cache 中的数据。

#### fsync 的 I/O 放大

每次 WAL fsync 产生至少 **两次 I/O**：
1. 数据写入（data block）
2. 元数据更新（inode 的 mtime/size 更新）

在 EXT4/XFS 上，metadata journal 可能产生额外 I/O。设置 `recycle_log_file_num` 可预分配 WAL 文件，减少元数据 I/O。

#### 量化影响

假设 NVMe fsync 延迟 100μs：
- **不开 Group Commit**：每次写入 100μs → 最大 10K writes/sec
- **Group Commit（10个写组合并）**：100μs / 10 = 10μs → 最大 100K writes/sec
- **禁用 WAL**：消除 fsync 瓶颈 → 写入性能提升 **2~5×**

### 瓶颈三：MemTable 写入的 CPU 竞争

默认 SkipList MemTable 的并发写入限制：

```cpp
// memtable.cc 中的插入路径
// SkipList 支持单写多读，不支持多写并发！
class SkipListRep : public MemTableRep {
  void Insert(KeyHandle handle) override {
    // InlineSkipList::Insert 使用 CAS 操作
    // 但同一位置的多个插入会产生 CAS 竞争
    skip_list_.Insert(static_cast<char*>(handle));
  }
};
```

开启 `allow_concurrent_memtable_write=true`（RocksDB 5.0+ 默认开启）后：
- 多个线程可以**并行插入 MemTable**
- 但 SkipList 的节点链接操作仍有 CAS 竞争
- 当 key 分布有热点时，竞争加剧

### 瓶颈四：Sequence Number 分配的全序要求

RocksDB 使用全局递增的 sequence number 保证一致性：

```cpp
// 每个写组在 mutex 保护下分配连续的 sequence number
// versions_impl.cc
uint64_t last_sequence = versions_->LastSequence();
WriteBatchInternal::SetSequence(merged_batch, last_sequence + 1);
versions_->SetLastSequence(last_sequence + total_count);
```

这个分配过程必须在 DB mutex 保护下完成，因为：
1. Sequence number 必须**全局唯一且递增**
2. Snapshot 读取依赖 sequence number 的可见性语义
3. 写组的原子性要求 sequence 连续分配

### 瓶颈五：Write Stall 的全局阻塞

当以下条件触发时，**所有**写入线程（包括所有 Column Family）都会被阻塞：

| 触发条件 | 默认阈值 | 行为 |
|---------|---------|------|
| Immutable MemTable 数量 | `max_write_buffer_number` (默认2) | **完全停止写入** |
| Level-0 文件数量 | `level0_slowdown_writes_trigger` (默认20) | 限速写入 |
| Level-0 文件数量 | `level0_stop_writes_trigger` (默认36) | **完全停止写入** |
| Pending Compaction 字节数 | `soft_pending_compaction_bytes` (默认64GB) | 限速写入 |
| Pending Compaction 字节数 | `hard_pending_compaction_bytes` (默认256GB) | **完全停止写入** |

**关键问题**：Write Stall 条件虽然是 per-Column Family 的，但 stall 行为影响整个 DB 实例。当 Compaction 跟不上写入速度时，所有线程都会被限速到 `delayed_write_rate`（默认 16MB/s）。

**Stall% 计算**：`Stall% = 累计 stall 时长 / 总运行时长`，由 compaction 统计输出（如 `Cumulative stall: 00:03:31, 65.1 percent`）。

**新版本 Stall 减少的原因**：Intra-L0 Compaction（L0 层内合并小 SST 降低 L0 文件数）、Compaction 调度与优先级优化、Subcompaction 分区更均匀、Compaction 输出对齐减少无效 I/O（7.8+）、以及相关 bug 修复，使 Flush/Compaction 更跟得上写入。

研究表明，在持续高写入负载下，**84.2% 的时间可能花在 compaction stall 上**。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-三、为什么-QPS-不随线程数线性增长.md]
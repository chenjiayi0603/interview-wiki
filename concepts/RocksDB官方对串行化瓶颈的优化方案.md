# RocksDB 官方对串行化瓶颈的优化方案

## 5.1 Pipelined Write（RocksDB 5.5+）

设计思路：将 WAL 写入和 MemTable 写入流水线化，让不同写组的这两个阶段并行执行。

```
默认模式（串行）:
  写组1: [WAL写入 | MemTable写入]
  写组2:                          [WAL写入 | MemTable写入]

Pipelined 模式（流水线）:
  写组1: [WAL写入 | MemTable写入     ]
  写组2:          [WAL写入 | MemTable写入     ]
  写组3:                   [WAL写入 | MemTable写入     ]
```

- 开启方式：`Options.enable_pipelined_write = true`
- 效果：8线程并发写场景下提升 **~30%** 吞吐
- 局限：WAL 写入本身仍是串行的，只是减少了等待时间

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-五、RocksDB-官方对串行化瓶颈的优化方案.md]

## 5.2 Unordered Write（RocksDB 6.3+）

放松写入顺序保证，允许多个写组的 WAL 和 MemTable 操作无序执行：

| 配置 | 保留的保证 | 吞吐提升 |
|------|-----------|---------|
| `unordered_write=true` + WritePrepared Txn | Read-your-own-writes + Immutable Snapshots | **34~42%** |
| `unordered_write=true`（单独使用） | 仅 Read-your-own-writes | **63~131%** |

代价：不再保证跨 batch 的原子读取语义和快照不可变性。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-五、RocksDB-官方对串行化瓶颈的优化方案.md]

## 5.3 Two Write Queues

将 WAL 写入和 MemTable 写入分成两个独立队列，减少 mutex 竞争：

```
单队列（默认）:
  DB mutex: [WAL + MemTable] → [WAL + MemTable] → ...

双队列:
  WAL 队列:      [WAL写入] → [WAL写入] → ...
  MemTable 队列:    [MemTable写入] → [MemTable写入] → ...
```

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-五、RocksDB-官方对串行化瓶颈的优化方案.md]

## 5.4 Manual WAL Flush

设置 `manual_wal_flush=true`，将 WAL 的 fwrite 操作推迟到用户显式调用 `DB::FlushWAL()`：
- 减少每次写入的 fwrite 系统调用开销
- MyRocks（MySQL on RocksDB）使用此特性获得 **40%** 吞吐提升
- 代价：crash 时可能丢失未 flush 的 WAL 数据

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-五、RocksDB-官方对串行化瓶颈的优化方案.md]

## 5.5 Concurrent MemTable Insert

`allow_concurrent_memtable_write=true`（5.0+ 默认开启）：
- Leader 完成 WAL 写入后，唤醒所有 follower 并行写入各自的 MemTable 部分
- 但 SkipList 的 CAS 竞争仍限制了实际并行度

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-五、RocksDB-官方对串行化瓶颈的优化方案.md]

## 5.6 各优化方案效果对比

| 优化方案 | 写入吞吐提升 | 适用场景 | 语义代价 |
|---------|------------|---------|---------|
| Pipelined Write | ~30% | WAL 开启的并发写 | 无 |
| Concurrent MemTable | ~15% | 多线程写入 | 无 |
| Manual WAL Flush | ~40% | 可接受短暂数据丢失 | 可能丢数据 |
| Unordered Write + WritePrepared | 34~42% | 事务型负载 | 需使用 TransactionDB |
| Unordered Write（relaxed） | 63~131% | 可接受弱一致性 | 无原子读/快照保证 |
| 禁用 WAL | 200~500% | 可从其他源恢复 | 无持久性保证 |

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-五、RocksDB-官方对串行化瓶颈的优化方案.md]
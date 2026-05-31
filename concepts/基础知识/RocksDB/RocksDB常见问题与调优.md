# RocksDB 常见问题与调优

## 写入停顿（Write Stall）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Write Stall 原因与解决                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  原因 1: L0 文件过多                                                         │
│  ─────────────────                                                          │
│  症状: level0_stop_writes_trigger 触发                                       │
│  解决:                                                                       │
│    - 增加 max_background_compactions                                        │
│    - 增加 max_subcompactions                                                │
│    - 增加 level0_stop_writes_trigger                                        │
│    - 使用 Universal Compaction                                              │
│                                                                             │
│  原因 2: Pending Compaction 字节过多                                         │
│  ──────────────────────────────                                             │
│  症状: soft/hard_pending_compaction_bytes_limit 触发                         │
│  解决:                                                                       │
│    - 增加 Compaction 线程数                                                  │
│    - 调高 pending_compaction_bytes_limit                                    │
│    - 使用 Rate Limiter 平滑 IO                                              │
│                                                                             │
│  原因 3: MemTable 过多                                                       │
│  ────────────────────                                                       │
│  症状: max_write_buffer_number 触发                                          │
│  解决:                                                                       │
│    - 增加 max_background_flushes                                            │
│    - 增加 max_write_buffer_number                                           │
│    - 减小 write_buffer_size                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-九、常见问题与调优.md]

## 线程竞争优化

```cpp
// 减少线程竞争的配置
rocksdb::Options ReduceContentionOptions() {
    rocksdb::Options options;
    
    // 1. 启用并发 MemTable 写入
    options.allow_concurrent_memtable_write = true;
    options.enable_write_thread_adaptive_yield = true;
    
    // 2. 多 Column Family 分散写入压力
    // 不同 CF 有独立的 MemTable 和 Flush
    
    // 3. 使用 WriteBatch 批量操作
    // 减少锁获取次数
    
    // 4. 调整写入线程 yield 参数
    options.write_thread_max_yield_usec = 100;
    options.write_thread_slow_yield_usec = 3;
    
    return options;
}
```

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-九、常见问题与调优.md]

## Related Pages
- [[RocksDB性能分析与瓶颈]]
- [[RocksDB故障场景与恢复测试]]
- [[RocksDB配置参数性能测试]]
- [[RocksDB文件体系]]
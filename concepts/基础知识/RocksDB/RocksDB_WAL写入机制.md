# RocksDB WAL 写入机制

## 六、WAL 写入机制

### 5.1 WAL 写入线程模型

```cpp
// WAL 写入配置
struct DBOptions {
    // WAL 目录
    std::string wal_dir;
    
    // WAL 大小限制
    uint64_t max_total_wal_size = 0;  // 0 表示自动计算
    
    // WAL 回收
    bool recycle_log_file_num = 0;
    
    // 手动 WAL Flush
    bool manual_wal_flush = false;
};

struct WriteOptions {
    // 是否同步写入 WAL
    bool sync = false;  // true: fsync, false: 仅 write
    
    // 是否禁用 WAL
    bool disableWAL = false;
};
```

**WAL 写入流程：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        WAL 写入流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   多个写入线程                                                       │
│   ┌────────┐ ┌────────┐ ┌────────┐                                  │
│   │Writer1 │ │Writer2 │ │Writer3 │                                  │
│   └───┬────┘ └───┬────┘ └───┬────┘                                  │
│       │          │          │                                       │
│       └──────────┼──────────┘                                       │
│                  │                                                  │
│                  ▼                                                  │
│   ┌──────────────────────────────────┐                              │
│   │       Write Batch Group          │  ← Leader 合并多个 batch     │
│   │   (Leader 线程负责 WAL 写入)      │                              │
│   └───────────────┬──────────────────┘                              │
│                   │                                                 │
│                   ▼                                                 │
│   ┌──────────────────────────────────┐                              │
│   │         WAL Writer               │                              │
│   │   1. 编码 WriteBatch             │                              │
│   │   2. 写入 log 文件               │                              │
│   │   3. 可选: fsync()               │                              │
│   └───────────────┬──────────────────┘                              │
│                   │                                                 │
│                   ▼                                                 │
│   ┌──────────────────────────────────┐                              │
│   │      WAL File (.log)             │                              │
│   │   ┌─────────────────────────┐    │                              │
│   │   │ Record 1 (WriteBatch)   │    │                              │
│   │   ├─────────────────────────┤    │                              │
│   │   │ Record 2 (WriteBatch)   │    │                              │
│   │   ├─────────────────────────┤    │                              │
│   │   │ ...                     │    │                              │
│   │   └─────────────────────────┘    │                              │
│   └──────────────────────────────────┘                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 WAL 同步策略

```cpp
// 不同同步策略的性能对比
void WALSyncStrategies() {
    rocksdb::WriteOptions write_options;
    
    // 策略 1: 不同步 (最快，但可能丢数据)
    write_options.sync = false;
    write_options.disableWAL = false;
    // 数据写入 OS buffer，由 OS 决定何时刷盘
    
    // 策略 2: 每次同步 (最安全，但最慢)
    write_options.sync = true;
    // 每次写入都 fsync，保证持久化
    
    // 策略 3: 禁用 WAL (最快，数据可能丢失)
    write_options.disableWAL = true;
    // 不写 WAL，崩溃时 MemTable 数据丢失
    
    // 策略 4: 组提交 (性能与安全平衡)
    // 通过 WriteOptions::no_slowdown 和批量写入实现
}
```

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-六、WAL-写入机制.md]
# RocksDB 线程配置最佳实践

## 7.1 通用配置建议

```cpp
#include "rocksdb/db.h"
#include "rocksdb/options.h"

rocksdb::Options GetOptimizedOptions() {
    rocksdb::Options options;
    
    // ==================== CPU 核心数相关配置 ====================
    int num_cores = std::thread::hardware_concurrency();
    
    // 后台线程总数建议: CPU 核心数的 1-2 倍
    options.max_background_jobs = num_cores;
    
    // 或者分别配置:
    // Flush 线程: 2-4 个通常足够
    options.env->SetBackgroundThreads(std::min(4, num_cores / 4 + 1), 
                                      rocksdb::Env::Priority::HIGH);
    // Compaction 线程: 可以更多
    options.env->SetBackgroundThreads(num_cores - 2, 
                                      rocksdb::Env::Priority::LOW);
    
    // ==================== 写入相关配置 ====================
    // 允许并发写入 MemTable
    options.allow_concurrent_memtable_write = true;
    
    // 启用流水线写入
    options.enable_pipelined_write = true;
    
    // 写入批量组大小
    options.write_batch_group_size_bytes = 1 << 20;  // 1MB
    
    // ==================== Compaction 相关配置 ====================
    // Compaction 并发子任务
    options.max_subcompactions = 4;  // 单个 Compaction 可并行执行
    
    // Level 0 阈值 (避免写入停顿)
    options.level0_file_num_compaction_trigger = 4;
    options.level0_slowdown_writes_trigger = 20;
    options.level0_stop_writes_trigger = 36;
    
    return options;
}
```

## 7.2 不同场景配置

```cpp
// 写入密集型场景
rocksdb::Options WriteHeavyOptions() {
    rocksdb::Options options;
    
    // 更大的 MemTable，减少 Flush 频率
    options.write_buffer_size = 256 << 20;  // 256MB
    options.max_write_buffer_number = 4;
    
    // 更多的 Compaction 线程
    options.max_background_jobs = 16;
    options.max_subcompactions = 8;
    
    // Universal Compaction (写放大更低)
    options.compaction_style = rocksdb::kCompactionStyleUniversal;
    
    return options;
}

// 读取密集型场景
rocksdb::Options ReadHeavyOptions() {
    rocksdb::Options options;
    
    // 更大的 Block Cache
    std::shared_ptr<rocksdb::Cache> cache = 
        rocksdb::NewLRUCache(2ULL << 30);  // 2GB
    rocksdb::BlockBasedTableOptions table_options;
    table_options.block_cache = cache;
    options.table_factory.reset(
        rocksdb::NewBlockBasedTableFactory(table_options));
    
    // Bloom Filter 减少磁盘读取
    table_options.filter_policy.reset(
        rocksdb::NewBloomFilterPolicy(10, false));
    
    // Level Compaction (读放大更低)
    options.compaction_style = rocksdb::kCompactionStyleLevel;
    
    return options;
}

// 低延迟场景
rocksdb::Options LowLatencyOptions() {
    rocksdb::Options options;
    
    // 较小的 MemTable，快速 Flush
    options.write_buffer_size = 64 << 20;  // 64MB
    
    // 避免写入停顿
    options.level0_slowdown_writes_trigger = 99999;
    options.level0_stop_writes_trigger = 99999;
    options.soft_pending_compaction_bytes_limit = 0;
    options.hard_pending_compaction_bytes_limit = 0;
    
    // Rate Limiter 限制后台 IO
    options.rate_limiter.reset(
        rocksdb::NewGenericRateLimiter(100 << 20));  // 100MB/s
    
    return options;
}
```

## 7.3 监控指标

```cpp
// 重要的线程相关监控指标
void PrintThreadMetrics(rocksdb::DB* db) {
    std::string value;
    
    // Flush 相关
    db->GetProperty("rocksdb.num-immutable-mem-table", &value);
    std::cout << "Immutable MemTables: " << value << std::endl;
    
    db->GetProperty("rocksdb.mem-table-flush-pending", &value);
    std::cout << "Pending Flush: " << value << std::endl;
    
    // Compaction 相关
    db->GetProperty("rocksdb.compaction-pending", &value);
    std::cout << "Pending Compaction: " << value << std::endl;
    
    db->GetProperty("rocksdb.num-running-compactions", &value);
    std::cout << "Running Compactions: " << value << std::endl;
    
    db->GetProperty("rocksdb.num-running-flushes", &value);
    std::cout << "Running Flushes: " << value << std::endl;
    
    // 写入停顿
    db->GetProperty("rocksdb.is-write-stopped", &value);
    std::cout << "Write Stopped: " << value << std::endl;
    
    // 线程池状态
    db->GetProperty("rocksdb.num-files-at-level0", &value);
    std::cout << "L0 Files: " << value << std::endl;
}
```

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-八、线程配置最佳实践.md]
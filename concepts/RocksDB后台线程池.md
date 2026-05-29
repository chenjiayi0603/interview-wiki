# RocksDB 后台线程池

## 5.1 线程池架构

RocksDB 使用两个优先级不同的线程池：

```cpp
// 线程池配置
#include "rocksdb/env.h"

void ConfigureThreadPool() {
    rocksdb::Options options;
    
    // HIGH 优先级线程池 - 用于 Flush
    // 默认线程数: 1
    options.env->SetBackgroundThreads(4, rocksdb::Env::Priority::HIGH);
    
    // LOW 优先级线程池 - 用于 Compaction
    // 默认线程数: 1
    options.env->SetBackgroundThreads(8, rocksdb::Env::Priority::LOW);
    
    // 或者通过 options 配置
    options.max_background_flushes = 4;      // Flush 线程数
    options.max_background_compactions = 8;  // Compaction 线程数
    
    // RocksDB 6.0+ 统一配置
    options.max_background_jobs = 12;        // 总后台线程数
}
```

**线程池结构：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                      RocksDB 后台线程池                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                 HIGH Priority Thread Pool                      │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │  │
│  │  │ Flush-1 │ │ Flush-2 │ │ Flush-3 │ │ Flush-4 │              │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘              │  │
│  │                                                                │  │
│  │  任务队列: [FlushJob, FlushJob, ...]                           │  │
│  │  特点: 高优先级, 快速响应, 防止写入停顿                          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                 LOW Priority Thread Pool                       │  │
│  │  ┌──────────┐┌──────────┐┌──────────┐┌──────────┐             │  │
│  │  │Compact-1 ││Compact-2 ││Compact-3 ││Compact-4 │ ...         │  │
│  │  └──────────┘└──────────┘└──────────┘└──────────┘             │  │
│  │                                                                │  │
│  │  任务队列: [CompactionJob, CompactionJob, ...]                 │  │
│  │  特点: 低优先级, CPU/IO 密集, 可延迟执行                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 5.2 Flush 线程

Flush 线程负责将 Immutable MemTable 写入 SST 文件：

```cpp
// Flush 触发条件
struct ColumnFamilyOptions {
    // MemTable 大小阈值，超过则触发 Flush
    size_t write_buffer_size = 64 << 20;  // 默认 64MB
    
    // 最大 Immutable MemTable 数量
    // 超过则阻塞写入等待 Flush
    int max_write_buffer_number = 2;
    
    // 触发 Flush 的最小 Immutable MemTable 数量
    int min_write_buffer_number_to_merge = 1;
};

// Flush 任务调度 (简化)
class DBImpl {
    void MaybeScheduleFlushOrCompaction() {
        // 检查是否需要 Flush
        for (auto cfd : *column_family_set_) {
            if (cfd->imm()->NumNotFlushed() > 0) {
                // 调度 Flush 任务到 HIGH 优先级线程池
                env_->Schedule(&DBImpl::BGWorkFlush, 
                              new FlushThreadArg(this, cfd),
                              Env::Priority::HIGH);
            }
        }
    }
    
    // Flush 后台工作函数
    static void BGWorkFlush(void* arg) {
        FlushThreadArg* fa = static_cast<FlushThreadArg*>(arg);
        fa->db->BackgroundFlush();
    }
};
```

**Flush 流程图：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Flush 流程                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Active MemTable                                                   │
│        │                                                            │
│        │ (写满 write_buffer_size)                                    │
│        ▼                                                            │
│   ┌────────────────────┐                                            │
│   │ 切换为 Immutable   │  ← 新建 Active MemTable                     │
│   │    MemTable        │                                            │
│   └─────────┬──────────┘                                            │
│             │                                                       │
│             │ (调度 Flush Job)                                       │
│             ▼                                                       │
│   ┌────────────────────┐                                            │
│   │ HIGH Priority Pool │                                            │
│   │   Flush Thread     │                                            │
│   └─────────┬──────────┘                                            │
│             │                                                       │
│             ▼                                                       │
│   ┌────────────────────────────────────────────────────┐            │
│   │                  FlushJob 执行                      │            │
│   │  1. 遍历 Immutable MemTable                        │            │
│   │  2. 构建 SST 文件 (TableBuilder)                   │            │
│   │  3. 写入 Data Blocks, Index Blocks, Filter Blocks  │            │
│   │  4. 更新 MANIFEST (版本信息)                        │            │
│   │  5. 删除 WAL (如果可以)                             │            │
│   └─────────────────────────────────────────────────────┘            │
│             │                                                       │
│             ▼                                                       │
│   ┌────────────────────┐                                            │
│   │  L0 SST 文件生成   │                                            │
│   └────────────────────┘                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 5.3 Compaction 线程

Compaction 是 RocksDB 最复杂的后台操作，负责合并 SST 文件：

```cpp
// Compaction 配置
struct ColumnFamilyOptions {
    // Compaction 风格
    CompactionStyle compaction_style = kCompactionStyleLevel;
    
    // Level 0 文件数阈值
    int level0_file_num_compaction_trigger = 4;  // 触发 Compaction
    int level0_slowdown_writes_trigger = 20;     // 开始限速写入
    int level0_stop_writes_trigger = 36;         // 停止写入
    
    // 各层大小配置
    uint64_t max_bytes_for_level_base = 256 << 20;  // L1 大小
    double max_bytes_for_level_multiplier = 10;     // 层间倍数
    
    // Compaction 线程限速
    uint64_t soft_pending_compaction_bytes_limit = 64ULL << 30;
    uint64_t hard_pending_compaction_bytes_limit = 256ULL << 30;
};

// Compaction 类型
enum CompactionReason {
    kUnknown,
    kLevelL0FilesNum,           // L0 文件数过多
    kLevelMaxLevelSize,         // 某层大小超限
    kUniversalSizeAmplification,// Universal: 空间放大
    kUniversalSizeRatio,        // Universal: 大小比例
    kUniversalSortedRunNum,     // Universal: sorted run 数量
    kFIFOMaxSize,               // FIFO: 超过最大大小
    kFIFOReduceNumFiles,        // FIFO: 减少文件数
    kFIFOTtl,                   // FIFO: TTL 过期
    kManualCompaction,          // 手动触发
    kFilesMarkedForCompaction,  // 文件被标记
    kBottommostFiles,           // 最底层文件
    kTtl,                       // TTL 过期
    kFlush,                     // Flush 触发
    kExternalSstIngestion,      // 外部 SST 导入
};
```

**Level Compaction 流程：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Level Compaction 示意图                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Level 0:  [SST] [SST] [SST] [SST] [SST]  ← 文件可能有 key 范围重叠         │
│               │     │     │     │     │                                     │
│               └─────┴─────┴─────┴─────┘                                     │
│                           │                                                 │
│                           │ (L0 → L1 Compaction)                            │
│                           ▼                                                 │
│  Level 1:  [SST-a] [SST-b] [SST-c] [SST-d]  ← key 范围不重叠                │
│                       │                                                     │
│                       │ (L1 → L2 Compaction)                                │
│                       ▼                                                     │
│  Level 2:  [SST] [SST] [SST] [SST] [SST] [SST] [SST] [SST]                  │
│                                                                             │
│  ...                                                                        │
│                                                                             │
│  Level N:  [SST] [SST] [SST] ... [SST]  ← 最大的一层                        │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Compaction 过程:                                                           │
│                                                                             │
│    输入文件                    输出文件                                      │
│    ┌───────┐                  ┌───────┐                                     │
│    │L1 SST │───┐              │新 SST │ (合并、去重、删除过期数据)           │
│    └───────┘   │   Merge      ├───────┤                                     │
│    ┌───────┐   ├─────────────►│新 SST │                                     │
│    │L2 SST │───┤   Sort       ├───────┤                                     │
│    └───────┘   │              │新 SST │                                     │
│    ┌───────┐   │              └───────┘                                     │
│    │L2 SST │───┘                  │                                         │
│    └───────┘                      ▼                                         │
│                              写入 Level 2                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Compaction 线程工作流程：**

```cpp
// Compaction Job 执行 (简化)
class CompactionJob {
public:
    Status Run() {
        // 1. 准备输入文件
        PrepareInputFiles();
        
        // 2. 创建迭代器，合并多个输入文件
        std::unique_ptr<InternalIterator> input(
            versions_->MakeInputIterator(compact_->compaction));
        
        // 3. 遍历并写入新 SST
        Status status;
        while (input->Valid() && !shutting_down_) {
            // 处理 key-value
            Slice key = input->key();
            Slice value = input->value();
            
            // 跳过已删除或过期的 key
            if (ShouldDrop(key, value)) {
                input->Next();
                continue;
            }
            
            // 写入输出文件
            status = builder_->Add(key, value);
            
            // 检查是否需要切换到新文件
            if (builder_->FileSize() >= max_output_file_size_) {
                status = FinishCompactionOutputFile();
                status = OpenCompactionOutputFile();
            }
            
            input->Next();
        }
        
        // 4. 完成输出文件
        status = FinishCompactionOutputFile();
        
        // 5. 安装 Compaction 结果 (更新版本)
        status = InstallCompactionResults();
        
        return status;
    }
};
```

## 5.4 Compaction 策略对比

```
┌────────────────┬────────────────────────┬────────────────────────┬─────────────────────┐
│     特性        │    Level Compaction    │  Universal Compaction  │   FIFO Compaction   │
├────────────────┼────────────────────────┼────────────────────────┼─────────────────────┤
│ 写放大          │ 较高 (~10-30x)         │ 较低 (~2-5x)           │ 最低 (1x)            │
├────────────────┼────────────────────────┼────────────────────────┼─────────────────────┤
│ 空间放大        │ 较低 (~10-20%)         │ 可能较高               │ 最低                 │
├────────────────┼────────────────────────┼────────────────────────┼─────────────────────┤
│ 读放大          │ 较低                   │ 较高                   │ 较高                 │
├────────────────┼────────────────────────┼────────────────────────┼─────────────────────┤
│ 适用场景        │ 通用场景               │ 写密集型               │ 时序数据/TTL 场景    │
├────────────────┼────────────────────────┼────────────────────────┼─────────────────────┤
│ 线程消耗        │ 中等                   │ 较低                   │ 最低                 │
└────────────────┴────────────────────────┴────────────────────────┴─────────────────────┘
```

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-五、后台线程池-五、后台线程池.md]
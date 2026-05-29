# RocksDB 前台线程（用户线程）

## 4.1 写入路径

用户线程执行写入操作时，数据流经以下路径：

```cpp
// 写入流程示例
#include "rocksdb/db.h"
#include "rocksdb/options.h"

void WriteExample() {
    rocksdb::DB* db;
    rocksdb::Options options;
    options.create_if_missing = true;
    
    rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
    
    // 用户线程直接执行写入
    // 1. 写入 WAL (如果启用)
    // 2. 写入 MemTable
    status = db->Put(rocksdb::WriteOptions(), "key1", "value1");
    
    // 批量写入 - 原子操作
    rocksdb::WriteBatch batch;
    batch.Put("key2", "value2");
    batch.Put("key3", "value3");
    batch.Delete("key1");
    status = db->Write(rocksdb::WriteOptions(), &batch);
    
    delete db;
}
```

**写入线程模型详解：**

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           写入请求处理流程（含串行化/加锁点）                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│    用户线程                                                                          │
│        │                                                                            │
│        ▼                                                                            │
│    ┌─────────────────────────────┐                                                  │
│    │ ① 获取写入锁 (JoinBatchGroup)│  ★ CAS 竞争 newest_writer_（串行化）             │
│    │   - Leader: 直接进入         │  ★ Follower: AwaitState 阻塞（mutex+cond/自旋）  │
│    └──────────────┬──────────────┘    源码: write_thread.cc LinkOne/AwaitState       │
│                   │                                                                │
│                   ▼                                                                │
│    ┌─────────────────────────────┐     ┌─────────────────────────────┐            │
│    │ ② 批量合并                  │────►│ ③ 写入 WAL                   │            │
│    │   Follower: LinkOne 链入    │     │   ★ 单文件顺序写（串行）      │            │
│    │   共享队列 newest_writer_    │     │   ★ sync 时 fsync 串行 I/O   │            │
│    │   Leader: 遍历队列并合并     │     │                              │            │
│    │   EnterAsBatchGroupLeader   │     │                              │            │
│    │   mutex_.Lock() 持有         │     │                              │            │
│    └──────────────┬──────────────┘     └──────────────┬──────────────┘            │
│                   │                                  │ 均在 mutex_ 保护下          │
│                   ▼                                  ▼                             │
│    ┌─────────────────────────────────────────────────────────────────┐            │
│    │ ④ 写入 MemTable                                                  │            │
│    │   默认: Leader 串行代写 | 并发模式: SkipList CAS 竞争（热点 key）  │            │
│    └──────────────┬──────────────────────────────────────────────────┘            │
│                   │                                                                │
│                   ▼                                                                │
│    ┌─────────────────────────────┐                                                  │
│    │ ⑤ 通知 Follower              │  ★ SetState: Writer::StateMutex + StateCV      │
│    │   ExitAsBatchGroupLeader    │  ★ notify_one() 唤醒等待线程                     │
│    └─────────────────────────────┘    源码: write_thread.cc SetState               │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**串行化与加锁点速览：**

| 步骤 | 串行化/加锁点 | 相关代码 |
|------|---------------|----------|
| ① JoinBatchGroup | CAS 竞争 `newest_writer_`；Follower 在 `AwaitState` 中阻塞 | `LinkOne()`, `AwaitState()`, `BlockingAwaitState()` |
| ② 批量合并 | Follower 链入共享队列；Leader 遍历并合并 | `LinkOne()`, `EnterAsBatchGroupLeader()` |
| ③ 写入 WAL | 单文件顺序写；fsync 串行 I/O | `WriteToWAL()`, `log_writer_->Sync()` |
| ④ 写入 MemTable | 默认 Leader 串行；并发模式 SkipList CAS | `InsertInto()`, `InlineSkipList::Insert()` |
| ⑤ 通知 Follower | `Writer::StateMutex` + `StateCV` | `SetState()`, `notify_one()` |

### 4.1.1 串行化与加锁点详解

#### ① JoinBatchGroup：CAS 串行化 + 条件变量等待

```cpp
// db/write_thread.cc
void WriteThread::JoinBatchGroup(Writer* w) {
  bool linked_as_leader = LinkOne(w, &newest_writer_);  // CAS 竞争，单点串行化
  
  if (!linked_as_leader) {
    // Follower 在此阻塞：自旋 200 次 → yield → mutex+cond
    AwaitState(w, STATE_GROUP_LEADER | STATE_MEMTABLE_WRITER_LEADER |
                  STATE_PARALLEL_MEMTABLE_CALLER | STATE_PARALLEL_MEMTABLE_WRITER |
                  STATE_COMPLETED, &jbg_ctx);
  }
}

// LinkOne: CAS 竞争 newest_writer_
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    w->link_older = writers;
    if (newest_writer->compare_exchange_weak(writers, w)) {
      return (writers == nullptr);  // writers==nullptr → 成为 Leader
    }
  }
}
```

#### ② 批量合并：Follower 链入队列 + Leader 遍历合并

**不是** Follower 提交到 Leader 的缓冲区，而是：

1. **Follower**：在 `JoinBatchGroup` → `LinkOne` 时，通过 CAS 把自己**链入共享队列**（`newest_writer_` 维护的链表），然后 `AwaitState` 阻塞等待。
2. **Leader**：在 `EnterAsBatchGroupLeader` 中**遍历**该共享队列（从自己到 `newest_writer_`），把兼容的 Writer 收集到 `write_group`，再合并 batch。

**Follower 链入共享队列的处理代码**（`db/write_thread.cc`）：

```cpp
// LinkOne: 通过 CAS 无锁链入 newest_writer_ 队列
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    w->link_older = writers;   // ① 设置自己的 link_older 指向当前队尾（即上一个 newest）
    // ② CAS: 若 newest_writer 仍为 writers，则原子替换为 w，否则重试
    if (newest_writer->compare_exchange_weak(writers, w)) {
      return (writers == nullptr);  // writers==nullptr 表示队列空，自己成为 Leader
    }
    // CAS 失败：其他线程抢先链入，writers 被更新为最新值，下一轮重试
  }
}
// 链入后队列形态: ... → link_older ← w (newest_writer_ 指向 w)
```

**Leader 遍历队列并收集**（`EnterAsBatchGroupLeader` 简化）：

```cpp
// Leader: EnterAsBatchGroupLeader 遍历队列并收集
Writer* w = leader;
while (w != newest_writer) {
  w = w->link_newer;
  // 检查兼容性: sync、no_slowdown、disable_wal、protection_bytes_per_key 等
  if (CompatibleWithLeader(w, leader)) {
    write_group->last_writer = w;
    write_group->size++;
    size += WriteBatchInternal::ByteSize(w->batch);
  }
}
```

**Leader 路径**（持 mutex 期间）：

```cpp
// db_impl_write.cc 简化流程
if (w.state == WriteThread::STATE_GROUP_LEADER) {
  mutex_.Lock();                              // 获取 DB 全局锁
  PreprocessWrite(...);                       // 检查 write stall
  write_thread_.EnterAsBatchGroupLeader(&w, &write_group);  // 遍历队列、合并
  status = WriteToWAL(write_group, ...);      // 写 WAL
  status = InsertInto(memtable, ...);         // 写 MemTable
  versions_->SetLastSequence(...);            // 更新 sequence
  mutex_.Unlock();                            // 释放锁
  write_thread_.ExitAsBatchGroupLeader(...);  // 唤醒 Follower
}
```

#### ③ 写入 WAL：天然串行

- WAL 为单文件追加，多线程并发写会破坏记录边界
- `sync=true` 时每次 fsync 为串行 I/O（NVMe 约 50~200μs）

#### ④ 写入 MemTable：取决于 allow_concurrent_memtable_write

| 配置 | 行为 |
|------|------|
| `false`（默认旧版） | Leader 代写整个 group，无额外锁 |
| `true`（RocksDB 5.0+ 默认） | 多线程并行插入，SkipList 内部 CAS 竞争 |

#### ⑤ 通知 Follower：SetState 中的锁

```cpp
// write_thread.cc
void WriteThread::SetState(Writer* w, uint8_t new_state) {
  if (state == STATE_LOCKED_WAITING || !w->state.compare_exchange_strong(state, new_state)) {
    std::lock_guard guard(w->StateMutex());
    w->state.store(new_state, std::memory_order_relaxed);
    w->StateCV().notify_one();  // 唤醒等待的 Follower
  }
}
```

### 4.2 写入批量合并（Write Batch Group）

RocksDB 使用 **WriteThread** 机制实现写入批量合并，提高写入吞吐：

```cpp
// WriteThread 核心机制 (简化版)
// 位于 db/write_thread.cc

class WriteThread {
public:
    // 写入者加入批量组
    // 第一个加入的成为 Leader，后续的成为 Follower
    void JoinBatchGroup(Writer* w);
    
    // Leader 完成后通知所有 Follower
    void ExitAsBatchGroupLeader(WriteGroup& write_group);
    
private:
    // 等待队列头指针
    std::atomic<Writer*> newest_writer_;
    
    // 批量组大小限制
    uint64_t max_write_batch_group_size_bytes_;
};

// Writer 状态
struct Writer {
    WriteBatch* batch;
    bool sync;               // 是否需要 sync
    bool no_slowdown;        // 是否允许延迟
    bool disable_wal;        // 是否禁用 WAL
    
    // 状态机
    enum State {
        STATE_INIT,
        STATE_GROUP_LEADER,      // 成为 Leader
        STATE_MEMTABLE_WRITER_LEADER,
        STATE_PARALLEL_MEMTABLE_WRITER,  // 并行写 MemTable
        STATE_COMPLETED          // 完成
    };
    std::atomic<State> state;
};
```

**批量合并流程：**

```
时间线 ─────────────────────────────────────────────────────────────►

Writer1 ─────┐
             │
Writer2 ─────┼──► JoinBatchGroup() ──► Leader (Writer1) 合并所有 batch
             │                              │
Writer3 ─────┘                              ▼
                                      ┌──────────────────┐
                                      │ 写入 WAL (合并后) │
                                      └────────┬─────────┘
                                               │
                          ┌────────────────────┼────────────────────┐
                          │                    │                    │
                          ▼                    ▼                    ▼
                    ┌──────────┐         ┌──────────┐         ┌──────────┐
                    │ Writer1  │         │ Writer2  │         │ Writer3  │
                    │ 写MemTable│         │ 写MemTable│         │ 写MemTable│
                    └──────────┘         └──────────┘         └──────────┘
                    (并行写入，如果启用 allow_concurrent_memtable_write)
```

### 4.3 读取路径

```cpp
// 读取流程
void ReadExample(rocksdb::DB* db) {
    std::string value;
    
    // 点查
    rocksdb::Status s = db->Get(rocksdb::ReadOptions(), "key1", &value);
    
    // 读取路径：
    // 1. MemTable (Active)
    // 2. Immutable MemTables
    // 3. Block Cache (如果命中)
    // 4. SST Files (L0 → L1 → ... → Ln)
}
```

**读取线程模型：**

```
┌────────────────────────────────────────────────────────────────────┐
│                        读取请求处理流程                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│    用户线程                                                         │
│        │                                                           │
│        ▼                                                           │
│    ┌─────────────────────────────────────────┐                     │
│    │         1. 查询 Active MemTable          │  ← 无锁并发读       │
│    │            (SkipList 实现)               │                     │
│    └────────────────┬────────────────────────┘                     │
│                     │ Miss                                         │
│                     ▼                                              │
│    ┌─────────────────────────────────────────┐                     │
│    │       2. 查询 Immutable MemTables        │  ← 从新到旧遍历     │
│    └────────────────┬────────────────────────┘                     │
│                     │ Miss                                         │
│                     ▼                                              │
│    ┌─────────────────────────────────────────┐                     │
│    │         3. 查询 Block Cache              │  ← LRU Cache (缓存的是 SST 文件中的 Data Block/Index Block)│
│    │            (Data Block / Index Block)   │                     │
│    └────────────────┬────────────────────────┘                     │
│                     │ Miss                                         │
│                     ▼                                              │
│    ┌─────────────────────────────────────────┐                     │
│    │         4. 查询 SST 文件                 │                     │
│    │    L0 (可能多个文件) → L1 → ... → Ln    │  ← 可能触发 I/O      │
│    └─────────────────────────────────────────┘                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

[src: raw/ingested/2技术/rocksdb/rocksdb的线程模型-四、前台线程（用户线程）-四、前台线程（用户线程）.md]
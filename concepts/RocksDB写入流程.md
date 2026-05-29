# RocksDB 写入流程

## 完整写入路径

```
1. 客户端调用 Put/Delete/Merge
   ↓
2. 封装为 WriteBatch（支持批量操作）
   ↓
3. 写入 WAL（Write-Ahead Log）
   - sync_log=true：强制同步磁盘（性能差）
   - sync_log=false：依赖 OS 缓存刷盘（推荐）
   ↓
4. 写入 MemTable（内存跳表）
   - 支持并发写入（allow_concurrent_memtable_write）
   - Leader-Follower 模型：合并多个 WriteBatch
   ↓
5. MemTable 满（write_buffer_size）
   - 转为 Immutable MemTable
   - 创建新 MemTable
   ↓
6. 后台 Flush 线程
   - Immutable MemTable → L0 SST 文件
   ↓
7. Compaction 触发
   - L0 → L1 → L2 ... → Ln
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## WriteBatch 批量写入

**WriteBatch 结构**：
```
[Sequence Number: 8 bytes]
[Count: 4 bytes]
[Data: Put/Delete/Merge 操作记录]
```

**批量写入优势**：
- **原子性**：一个 WriteBatch 内的操作要么全部成功，要么全部失败
- **性能**：减少 WAL 写入次数，降低 IO 开销
- **事务支持**：事务操作封装为 WriteBatch

**使用示例**：
```cpp
WriteBatch batch;
batch.Put("key1", "value1");
batch.Put("key2", "value2");
batch.Delete("key3");
db->Write(WriteOptions(), &batch);
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## WAL 预写日志机制

**WAL 作用**：
- **持久性保证**：崩溃恢复时重建 MemTable
- **顺序写入**：追加写入，性能高
- **原子性**：一个 WriteBatch 对应一条 WAL 记录

**WAL 文件格式**：
```
[Record Length: 4 bytes]
[Sequence Number: 8 bytes]
[Record Type: 1 byte]  // PUT/DELETE/MERGE
[Key Length: varint]
[Key Data]
[Value Length: varint]  // DELETE 无 Value
[Value Data]
[CRC32: 4 bytes]
```

**WAL 配置参数**：
- **sync_log**：
  - true：每次写入同步磁盘（性能差，数据安全）
  - false：依赖 OS 缓存刷盘（推荐，性能好）
- **max_total_wal_size**：WAL 总大小限制，超过触发切换
- **wal_dir**：WAL 文件目录

**WAL 切换时机**：
1. WAL 大小超过 max_total_wal_size
2. MemTable 切换（Flush 触发）
3. 手动 Flush（FlushWAL）

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## MemTable 写入优化

**并发写入机制**：
- **Leader-Follower 模型**：
  - 多个写入线程组成 Group
  - 第一个线程成为 Leader，负责 WAL 写入
  - Follower 等待 Leader 唤醒后并发写入 MemTable
- **配置**：allow_concurrent_memtable_write = true

**流水线写入**：
- **原理**：第一个 Group 的 MemTable 写入时，第二个 Group 可开始 WAL 写入
- **配置**：enable_pipelined_write = true
- **性能提升**：sync0 模式 3 倍，sync1 模式 2 倍

**MemTable 切换流程**：
```
MemTable 满（write_buffer_size）
→ 创建新 MemTable
→ 当前 MemTable 转为 Immutable
→ 调度后台 Flush 线程
→ Immutable MemTable → L0 SST
```

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Flush 机制

**Flush 触发条件**：
1. MemTable 写满（write_buffer_size）
2. 手动 Flush（Flush）
3. WAL 大小超限（max_total_wal_size）

**Flush 流程**：
```
1. 选择 Immutable MemTable
2. 创建 L0 SST 文件
3. 遍历 MemTable 写入 SST
4. 构建 Index Block 和 Filter Block
5. 写入 Footer
6. 更新 Manifest
7. 删除 Immutable MemTable
```

**Flush 性能优化**：
- **多线程 Flush**：max_background_flushes > 1
- **压缩**：L0 可禁用压缩减少 CPU 开销
- **限流**：避免 Flush 占用过多 IO 资源

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Write Stall 写入限流

**触发条件**：
- Immutable MemTable 数量 > max_write_buffer_number（默认 2）
- L0 文件数 > level0_slowdown_writes_trigger（默认 20）
- Pending Compaction Bytes > soft_pending_compaction_bytes_limit

**限流机制**：
- **延迟写入**：写入线程 sleep，降低写入速度
- **阻塞写入**：完全停止写入，等待 Compaction 完成
- **日志记录**：记录 write stall 信息，便于监控

**优化建议**：
- 增加后台 Compaction 线程
- 调整 Compaction 策略
- 监控 Compaction 延迟

[src: raw/ingested/2技术/rocksdb/RocksDB复习文档.md]

## Related Pages
- [[RocksDB概述]]
- [[RocksDB LSM-Tree]]
- [[RocksDB读取流程]]
- [[RocksDB Compaction]]
- [[RocksDB性能调优]]

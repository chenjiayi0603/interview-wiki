# KeyDB 性能优势总结

> 本文档总结 KeyDB 相对于 Redis 的性能优势来源、适用场景、核心结论，以及与 RocksDB 的写路径对比。

See also: [[KeyDB存算分离项目]], [[RocksDB性能分析与瓶颈]], [[Redis性能问题]]

---

## 一、对比 Redis 的性能优势来源

| 优化维度 | Redis | KeyDB | 性能增益来源 |
|----------|-------|-------|-------------|
| 事件循环 | 单线程 | 多线程 | CPU 多核利用 |
| 网络 I/O | 单线程（v6 可多线程） | 多线程 | 并行读写 |
| 命令解析 | 单线程 | 多线程 | 并行解析 |
| 命令执行 | 单线程 | 多线程（短持锁） | 并行执行 |
| accept() | 单线程 | SO_REUSEPORT 多线程 | 消除 accept 瓶颈 |
| 锁机制 | 无需（单线程） | 汇编级 FastLock | 纳秒级同步 |
| CPU 亲和性 | 无 | 线程绑核 | 缓存命中率提升 |
| 读操作 | 串行 | MVCC 无锁读 | 读零竞争 |
| 后台保存 | fork() COW | 进程内 MVCC 快照 | 更低内存开销 |
| Rehash | 阻塞式渐进 rehash | 自旋等待时 async rehash | 隐藏 rehash 开销 |

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-七、KeyDB-性能优势总结.md]

---

## 二、适用场景建议

### KeyDB 更优的场景

- 多核服务器（8+ 核心），充分发挥多线程优势
- 高并发连接数（100+ 并行连接）
- 读多写少的缓存场景（MVCC 无锁读优势巨大）
- 希望用单节点替代 Redis 集群以简化运维
- 需要 Active-Replica 双主架构

### Redis 仍有优势的场景

- 超大 value 场景（> 1MB），网络带宽成为主要瓶颈而非 CPU
- 对数据一致性要求极高（不接受 MVCC 的最终一致性模型）
- 生态系统依赖，需要最新 Redis 特性或模块
- 低核心数环境（2 核以下差距不大）

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-七、KeyDB-性能优势总结.md]

---

## 三、核心结论

KeyDB 的高性能**不是**来自某个单一优化，而是**多层次架构改进的叠加效果**：

1. **对称多线程**让 CPU 多核利用率接近线性扩展
2. **SO_REUSEPORT** 消除了网络层的线程竞争
3. **汇编级 FastLock** 将同步开销降到纳秒级
4. **极短的临界区**使全局锁不会成为瓶颈（锁利用率仅 ~3%）
5. **MVCC 无锁读**让读操作完全并行
6. **Async Rehash** 将自旋等待的 CPU 周期转化为有用工作
7. **线程绑核**提升了 CPU 缓存命中率
8. **进程内快照保存**替代了 fork()，减少了内存开销和系统调用

这些优化相互配合，使得 KeyDB 在多核硬件上实现了接近理论上限的并行效率。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-七、KeyDB-性能优势总结.md]

---

## 四、QPS 随线程线性增长的根本原因

| 因素 | 说明 |
|------|------|
| **串行化占比极低** | 持锁时间仅占单次请求的 ~0.4%–1.6%（20–50ns vs 3–12μs），阿姆达尔定律下可并行部分占主导 |
| **无共享写队列** | 每个线程直接执行写命令，无中心队列，无排队延迟 |
| **锁竞争概率低** | 典型场景下锁竞争概率 ~3%，多线程几乎互不阻塞 |
| **临界区极短** | 仅 dictAdd/dictFind 等 O(1) 操作在锁内，无 I/O、无大计算 |
| **网络不串行，AOF 落盘串行但不阻塞** | 网卡 RX/TX、Socket 缓冲、epoll 为 per-CPU/per-thread；AOF 落盘（write+fsync）虽串行（单线程写、单 BIO 线程刷盘），但 write→Page Cache 不阻塞、fsync 异步，Worker 不等待，故不成瓶颈 |

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-七、KeyDB-性能优势总结.md]

---

## 五、串行化非常少的具体原因

1. **命令执行层**：`AeLocker::arm()` 仅在 `processCommand` 前后持锁，锁内仅做 dict 修改 + `feedAppendOnlyFile`（纯内存 memcpy）+ `replicationFeedSlaves`，总耗时 ~100–800ns。
2. **网络层**：`readQueryFromClient` 标记 `AE_READ_THREADSAFE`，读和解析全程无锁；`handleClientsWithPendingWrites` 使用 per-thread 的 `clients_pending_write`，无全局串行点。
3. **持久化层**：`feedAppendOnlyFile` 只追加到内存 `aof_buf`；`write()` 进 Page Cache；`fsync()` 由 BIO 线程异步执行，`everysec` 策略下可推迟，不阻塞 Worker。
4. **内存分配**：jemalloc 线程本地缓存，减少全局锁竞争。
5. **读写分离**：读走 MVCC 无锁路径，写走短持锁路径，两者互不串行化。

**结论**：KeyDB 通过「最小化临界区 + 无锁读 + 并行 I/O + 无共享队列」的设计，让串行化几乎只发生在「修改 dict 的几十纳秒」内，因此 QPS 能随线程数线性扩展。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-七、KeyDB-性能优势总结.md]

---

## 六、与 RocksDB 对比：为何 KeyDB 无 Write Stall、写路径串行化极少

| 维度 | RocksDB | KeyDB |
|------|---------|-------|
| **Write Stall** | 有：compaction 跟不上时 L0 堆积、memtable 满，所有写线程被限速 | 无：纯 append-only 日志，无 compaction、无 L0、无 memtable 竞争 |
| **写路径串行化** | ~60%：WAL fwrite 串行、memtable 串行、sequence 全局、Leader 独占 | ~0.4%–1.6%：仅 dict 修改 + aof_buf 追加在锁内，~100–800ns |
| **AOF/WAL 的 write()** | WAL fwrite 在 Leader 的写路径内，阻塞直到进 Page Cache | AOF write() 在 beforeSleep 中，不在 processCommand 的临界区内 |
| **请求路径是否包含磁盘写** | 是：Leader 的 Put 路径包含 WAL fwrite（同步到 Page Cache） | 否：Worker 只做 dict + feedAppendOnlyFile（内存），write() 由主线程在事件循环中执行 |

### 为何 AOF 的 write() 虽同步却不造成 Write Stall？

1. **请求路径与落盘路径解耦**：Worker 在持锁期间只做 `feedAppendOnlyFile`（memcpy 到 aof_buf），锁释放后请求即完成。`write(aof_fd)` 在 `beforeSleep` 中执行，与具体请求不在同一调用栈。
2. **write() 只等到 Page Cache**：与 RocksDB 的 WAL fwrite 类似，阻塞到数据进 Page Cache 即返回（~1–5μs），不等待磁盘。
3. **无 compaction 背压**：RocksDB 的 Write Stall 来自 L0 过多、memtable 无法 flush。KeyDB 无此类结构，AOF 只追加，无合并、无重组。
4. **fsync 异步**：`everysec` 下 fsync 由 BIO 线程执行，Worker 不等待；`always` 下才会在 flush 时同步 fsync，此时才可能成为瓶颈。

### 为何 KeyDB 写路径串行化极少？

1. **对称多线程 vs Leader-Follower**：RocksDB 仅 Leader 做 WAL+memtable 写入；KeyDB 所有 Worker 均可执行命令，仅用短持锁保护 dict 和 aof_buf。
2. **临界区内无 I/O**：锁内只有 dictAdd、memcpy、replicationFeedSlaves，无 write()、无 fsync()。
3. **无全局 sequence**：RocksDB 的 sequence 分配是全局串行点；KeyDB 无此概念。
4. **Amdahl 定律**：RocksDB 串行比例 s≈60%，理论上限 ~1.67×；KeyDB 串行比例 s≈1%，理论上限接近 N×。

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-七、KeyDB-性能优势总结.md]

## Related Pages
- [[KeyDB存算分离项目]]
- [[RocksDB性能分析与瓶颈]]
- [[Redis性能问题]]
- [[Redis存储型方案性能对比]]
- [[OBS对接RocksDB性能分析]]
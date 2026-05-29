# RocksDB 基础

> RocksDB 是 Facebook 基于 LevelDB 开发的嵌入式持久化 KV 存储引擎，针对 SSD 和高速存储优化，广泛应用于数据库、流处理和缓存系统。

See also: [[OBS对接RocksDB性能分析]], [[KeyDB存算分离项目]]

---

## 一、核心架构

### 1.1 LSM-Tree 存储结构

RocksDB 基于 LSM-Tree（Log-Structured Merge-Tree）架构：

```
写入路径：
  SET(key, value)
    → 1. 写 WAL（Write-Ahead Log，预写日志）
    → 2. 写 MemTable（内存中的 SkipList）
    → 3. 返回客户端 OK

后台路径：
    → 4. MemTable 满 → 转为 Immutable MemTable
    → 5. Flush → 写入 L0 SST 文件（磁盘）
    → 6. Compaction → 合并多层 SST 文件
```

### 1.2 核心组件

| 组件 | 作用 | 存储位置 |
|------|------|---------|
| **MemTable** | 内存写入缓冲区（SkipList） | 内存 |
| **Immutable MemTable** | 只读的 MemTable，等待 Flush | 内存 |
| **WAL（Write-Ahead Log）** | 预写日志，保证持久性 | 磁盘 |
| **SST（Sorted String Table）** | 有序的持久化数据文件 | 磁盘 |
| **Block Cache** | 缓存热点数据块 | 内存 |
| **Bloom Filter** | 快速判断 key 是否存在 | 内存 |
| **MANIFEST** | 记录所有 SST 文件的元数据 | 磁盘 |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 二、Compaction 策略

### 2.1 Level Compaction

- 数据按层级组织，L0 层 SST 可重叠，L1+ 层 SST 按 key range 有序且不重叠
- 写放大：10~30×（不含 WAL）
- 读放大：低（每层最多 1 个 SST 覆盖同一 key range）
- 适合：读多写少

### 2.2 Universal Compaction

- 所有 SST 按时间顺序排列，同层内 SST 可重叠
- 写放大：1.3~2.5×（不含 WAL）
- 读放大：高（点查需遍历 10~30+ SST）
- 适合：写多读少

### 2.3 对比

| 维度 | level compaction | universal compaction |
|------|-----------------|---------------------|
| 写放大 | 10~30× | 1.3~2.5× |
| 读放大 | 低（每层最多 1 SST） | 高（点查需遍历 10~30+ SST） |
| 冷 GET 的 IO 次数 | 5~15 次 | 20~60 次 |
| 空间放大 | 较低 | 较高 |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 三、读路径

### 3.1 读流程

```
GET(key):
  1. 查 Active MemTable              → 内存，0 次磁盘 IO
  2. 查 Immutable MemTable(s)        → 内存，0 次磁盘 IO
  3. 查 L0 SST（N 个）:
     每 SST:
       a. 读 Filter Block（Bloom Filter）→ 1 次磁盘 IO
       b. 若 Bloom 判可能存在:
          - 读 Index Block           → 1 次磁盘 IO
          - 读 Data Block            → 1 次磁盘 IO
  4. 查 L1~Ln（每层最多 1 SST）:
     同上 a+b+c
```

### 3.2 读放大

- level compaction：一次 GET 可能触发 5~15 次磁盘 IO
- universal compaction：一次 GET 可能触发 20~60 次磁盘 IO
- Block Cache 命中时：0 次磁盘 IO

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 四、写路径

### 4.1 写流程

```
SET(key, value):
  1. 写 WAL（sync fsync 到磁盘）
  2. 写 MemTable（内存 SkipList）
  3. 返回客户端 OK

后台：
  4. MemTable 满 → Flush → 写 L0 SST
  5. Compaction → 合并多层 SST
```

### 4.2 WAL 策略

| 策略 | 持久性 | 延迟 |
|------|--------|------|
| `sync_log=true`（每次 fsync） | 强 | 高（0.1~30ms/次，取决于存储介质） |
| `sync_log=false`（OS buffer） | 弱（崩溃可能丢） | 低（内存级） |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 五、Write Stall 机制

RocksDB 的 write stall 机制防止写入速度超过 Compaction 处理能力：

1. L0 文件数超过 `level0_slowdown_writes_trigger`（默认 20）→ 每次写入强制 sleep 1ms
2. L0 文件数超过 `level0_stop_writes_trigger`（默认 36）→ **完全停止接受写入**
3. Immutable MemTable 积压超过 `max_write_buffer_number` → write stop

在慢存储（如 OBS）上，Compaction 耗时数秒~数十秒，stall 频率和持续时间是本地 SSD 的 10~50 倍。

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 六、Env 抽象层

RocksDB 通过 `Env` 接口抽象文件系统操作，支持自定义存储后端：

```cpp
class Env {
public:
    virtual Status NewWritableFile(const std::string& fname, ...) = 0;
    virtual Status NewSequentialFile(const std::string& fname, ...) = 0;
    virtual Status NewRandomAccessFile(const std::string& fname, ...) = 0;
    virtual Status DeleteFile(const std::string& fname) = 0;
    virtual Status CreateDir(const std::string& dirname) = 0;
    virtual Status GetChildren(const std::string& dir, ...) = 0;
};
```

通过继承 `EnvWrapper` 并重写这些方法，可以将 RocksDB 的底层存储从本地文件系统替换为 OBS/S3 等远程对象存储。

[src: raw/ingested/3项目/KeyDB-Redis-华为云/面试故事-KeyDB.md]

## Related Pages
- [[OBS对接RocksDB性能分析]]
- [[KeyDB存算分离项目]]
- [[OBS对象存储]]

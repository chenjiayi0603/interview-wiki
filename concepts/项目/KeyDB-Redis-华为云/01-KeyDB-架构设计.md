# KeyDB 存算分离架构设计

> RocksDB Env 插件、存算分离、元数据中心（MySQL vs Etcd 选型分析）、文件类型管理。

---

## 一、存算分离架构

```
┌────────────────────────────────────────────────┐
│              KeyDB 进程                          │
│  ┌────────────────────────────────────────┐    │
│  │          KeyDB 核心逻辑                  │    │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │    │
│  │  │命令解析│ │数据结构│ │多线程 │ │网络层 │ │    │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ │    │
│  └───────────────┬────────────────────────┘    │
│                  ▼                               │
│  ┌────────────────────────────────────────┐    │
│  │          RocksDB（动态库）               │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐     │    │
│  │  │MemTable│ │SSTable │ │Compaction│    │    │
│  │  │(内存表) │ │(磁盘文) │ │(压缩合并)│    │    │
│  │  └────────┘ └────────┘ └────────┘     │    │
│  └───────────────┬────────────────────────┘    │
│                  ▼                               │
│  ┌────────────────────────────────────────┐    │
│  │       自定义 Env 插件（核心）            │    │
│  │  ┌──────────┐ ┌────────┐ ┌──────────┐ │    │
│  │  │ 文件操作  │ │元数据  │ │远程存储   │ │    │
│  │  │ 接口     │ │中心    │ │(OBS)     │ │    │
│  │  └──────────┘ └────────┘ └──────────┘ │    │
│  └────────────────────────────────────────┘    │
└──────────────────────┬─────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                              │
   ┌────▼────┐                   ┌────▼────┐
   │ 元数据  │                   │ 对象存储 │
   │(MySQL/ │                   │(OBS/S3) │
   │ Etcd)  │                   │         │
   └─────────┘                   └─────────┘
```

## 二、RocksDB Env 接口与文件系统抽象

### 2.1 Env 类层次结构

RocksDB 的 Env（Environment）是一个抽象基类，封装了所有与操作系统交互的操作。核心设计目标：**插件化存储后端**，所有文件 I/O 都通过 Env 接口完成。

```
┌─────────────────────────────────┐
│   Env（抽象基类）                │  ← RocksDB 定义的纯虚接口
│   virtual NewWritableFile()     │
│   virtual NewRandomAccessFile() │
│   virtual ...                   │
├─────────────────────────────────┤
│         EnvWrapper              │  ← 代理模式，转发调用到目标 Env
│     (继承 Env + 组合一个 Env)   │     方便用户只重写关心的接口
├─────────────────────────────────┤
│   DefaultEnv (Posix/Win32)      │  ← 本地文件系统实现（默认）
├─────────────────────────────────┤
│   CustomEnv                     │  ← 用户自定义实现
│   (HDFSEnv / OBSEnv / S3Env)   │     存算分离的核心扩展点
└─────────────────────────────────┘
```

关键关系：

| 类 | 说明 |
|---|------|
| `Env` | RocksDB 定义的抽象基类，约 40+ 个纯虚函数 |
| `EnvWrapper` | 继承 Env，内部持有另一个 Env 指针，默认转发所有调用；用户只需重写关心的少数方法 |
| `PosixEnv / Win32Env` | 对应操作系统的默认实现，`Env::Default()` 返回 |
| 自定义 Env | 继承 EnvWrapper，重写文件操作接口，实现存储重定向 |

### 2.2 Env 核心接口总览

#### 文件创建/打开

```cpp
class Env {
 public:
  // 创建可写文件（写入 SSTable / WAL / MANIFEST 等）
  virtual Status NewWritableFile(
      const std::string& fname,
      std::unique_ptr<WritableFile>* result,
      const EnvOptions& options) = 0;

  // 创建随机读文件（读取 SSTable 块）
  virtual Status NewRandomAccessFile(
      const std::string& fname,
      std::unique_ptr<RandomAccessFile>* result,
      const EnvOptions& options) = 0;

  // 创建顺序读文件（读取 WAL / MANIFEST 回放）
  virtual Status NewSequentialFile(
      const std::string& fname,
      std::unique_ptr<SequentialFile>* result,
      const EnvOptions& options) = 0;

  // 打开目录
  virtual Status NewDirectory(
      const std::string& name,
      std::unique_ptr<Directory>* result) = 0;
};
```

#### 文件元数据与目录操作

```cpp
  // ─── 文件存在性 & 枚举 ───
  virtual Status FileExists(const std::string& fname) = 0;
  virtual Status GetChildren(const std::string& dir,
                              std::vector<std::string>* result) = 0;
  virtual Status GetChildrenFileAttributes(
      const std::string& dir,
      std::vector<FileAttributes>* result) = 0;

  // ─── 文件信息 ───
  virtual Status GetFileSize(const std::string& fname,
                              uint64_t* file_size) = 0;
  virtual Status GetFileModificationTime(
      const std::string& fname,
      uint64_t* file_mtime) = 0;
  virtual Status GetAbsolutePath(
      const std::string& db_path,
      std::string* output_path) = 0;
```

#### 文件系统操作

```cpp
  // ─── 文件操作 ───
  virtual Status RenameFile(const std::string& src,
                             const std::string& target) = 0;
  virtual Status LinkFile(const std::string& src,
                           const std::string& target) = 0;
  virtual Status DeleteFile(const std::string& fname) = 0;

  // ─── 目录操作 ───
  virtual Status CreateDir(const std::string& dirname) = 0;
  virtual Status CreateDirIfMissing(const std::string& dirname) = 0;
  virtual Status DeleteDir(const std::string& dirname) = 0;

  // ─── 文件锁（跨进程互斥）───
  virtual Status LockFile(const std::string& fname,
                           FileLock** lock) = 0;
  virtual Status UnlockFile(FileLock* lock) = 0;
```

#### 线程池与调度

```cpp
  // ─── 后台线程（Flush / Compaction 线程池）───
  virtual void SetBackgroundThreads(int number,
                                     Priority pri) = 0;
  virtual int GetBackgroundThreads(Priority pri) = 0;
  virtual void Schedule(void (*function)(void* arg),
                         void* arg, Priority pri,
                         void* tag = nullptr,
                         void (*unschedFunction)(void* arg) = nullptr) = 0;
};
```

### 2.3 文件类型枚举（FileType）

RocksDB 内部用 `FileType` 枚举标记每种文件，用于决定不同的处理策略：

```cpp
enum FileType {
  kWalFile,              // WAL 日志         后缀 .log
  kDBWalFile,            // 数据库内部 WAL    后缀 .log
  kTableFile,            // SSTable 数据文件  后缀 .ldb / .sst
  kDescriptorFile,       // MANIFEST 清单     后缀 .manifest
  kCurrentFile,          // CURRENT 指针      无后缀
  kTempFile,             // 临时文件          后缀 .dbtmp
  kInfoLogFile,          // RocksDB 运行日志  后缀 .log
  kOptionsFile,          // 持久化配置        后缀 .options
  kIdentityFile,         // DB 实例标识       文件名 IDENTITY
  kLockFile,             // 进程锁            文件名 LOCK
};
```

各文件类型的生命周期与存储特性：

| 文件类型 | 生成时机 | 删除时机 | 数据量 | 访问模式 | 一致性要求 |
|---------|---------|---------|-------|---------|-----------|
| **kTableFile** | Compaction / Flush | 下一轮 Compaction | 最大（GB~TB） | 读多写少，随机读 | 最终一致 |
| **kWalFile** | 每次写操作 | Flush 完成 | 中等 | 顺序写，恢复时顺序读 | 强一致 |
| **kDescriptorFile** | 每次元数据变更 | 被新 MANIFEST 替换 | 小 | 顺序写，启动时顺序读 | 强一致 |
| **kCurrentFile** | MANIFEST 切换时 | 下次切换时 | 极小 | 写入少，读取频繁 | 强一致 |
| **kOptionsFile** | 配置变更 | 下次配置变更 | 极小 | 极少访问 | 最终一致 |
| **kLockFile** | DB 打开时 | DB 关闭时 | 极小 | 打开时锁操作 | 强一致 |

### 2.4 文件接口详解

#### WritableFile —— 顺序写入接口

所有 RocksDB 的写入操作（SSTable 构建、WAL 追加、MANIFEST 更新）都通过 `WritableFile`：

```cpp
class WritableFile {
 public:
  // ─── 核心写入 ───
  virtual Status Append(const Slice& data) = 0;              // 追加数据块
  virtual Status Append(const Slice& data,                   // 带校验的追加
                         const DataVerificationInfo& info);
  virtual Status PositionedAppend(const Slice& data,         // 指定偏移写入
                                   uint64_t offset);
  // ─── 生命周期 ───
  virtual Status Close() = 0;                                // 关闭文件
  virtual Status Flush() = 0;                                // 刷到 OS 缓冲区
  virtual Status Sync() = 0;                                 // fsync 落盘

  // ─── 大小 & 截断 ───
  virtual uint64_t GetFileSize() = 0;                        // 当前写入大小
  virtual Status Truncate(uint64_t size);                    // 截断到指定大小
  virtual Status Truncate();

  // ─── 缓存控制 ───
  virtual Status InvalidateCache(uint64_t offset,             // 缓存失效
                                  uint64_t length);
  virtual Status RangeSync(uint64_t offset,                   // 范围同步
                            uint64_t nbytes);
  virtual bool IsSyncThreadSafe() const;                     // 是否支持并发 Sync

  // ─── 性能统计 ───
  virtual size_t GetRequiredBufferAlignment() const;         // 对齐要求
};
```

写入生命周期：

```
RocksDB 层           WritableFile            存储后端
  │                     │                      │
  ├─ NewWritableFile ──→│←—— 创建文件句柄 ——→│
  ├─ Append(data1) ────→│——— 追加缓冲区 ———→│  (可批量)
  ├─ Append(data2) ────→│                      │
  ├─ Flush() ──────────→│——— flush → OS ———→│
  ├─ Sync() ───────────→│——— fsync → 持久化 →│
  ├─ Append(data3) ────→│                      │
  ├─ Close() ──────────→│——— 写入尾部 + 关闭 →│
  │                     │                      │
```

#### RandomAccessFile —— 随机读取接口

SSTable 的查询读取全部通过 `RandomAccessFile`：

```cpp
class RandomAccessFile {
 public:
  // ─── 核心读取（必须实现）───
  // 从 offset 处读取 n 字节，结果填入 scratch，返回 Slice
  virtual Status Read(uint64_t offset, size_t n,
                       Slice* result, char* scratch) const = 0;

  // ─── 预读取（可选）───
  virtual Status Prefetch(uint64_t offset, size_t n);

  // ─── 直接 IO ───
  virtual size_t GetUniqueId(char* id, size_t max_size) const;
  virtual bool ShouldForwardRawRequest() const;

  // ─── 内存映射（mmap 优化）───
  // 当文件被映射到内存时，RocksDB 可以直接拿到指针
  struct ReadResult {
    Status status;
    size_t size;
    const char* data;   // 指向 scratch 或 mmap 区域
  };
};
```

读取路径示意：

```
Get(key)
  → BlockBasedTable::Get()
    → RandomAccessFile::Read(offset, block_size)
      → 本地缓存命中 ? 直接返回 : 从远程 OBS 读取
    → 解析 Block（索引 / 布隆过滤器 / 数据块）
  → 返回 value
```

关键设计点：`scratch` 缓冲区由调用者提供，避免 `RandomAccessFile` 内部反复分配内存；返回的 `Slice` 可能指向 `scratch` 也可能指向 mmap 区域（如果支持）。

#### SequentialFile —— 顺序读取接口

WAL 恢复、MANIFEST 回放等场景使用顺序读：

```cpp
class SequentialFile {
 public:
  // ─── 顺序读取 ───
  virtual Status Read(size_t n, Slice* result,
                       char* scratch) = 0;

  // ─── 跳过指定字节 ───
  virtual Status Skip(uint64_t n) = 0;

  // ─── 带偏移的顺序读（可选项）───
  virtual Status PositionedRead(uint64_t offset, size_t n,
                                 Slice* result, char* scratch);
};
```

与 `RandomAccessFile` 的区别：

| 特性 | SequentialFile | RandomAccessFile |
|------|---------------|-----------------|
| 访问模式 | 从头到尾顺序 | 任意偏移随机 |
| 内部缓冲 | 支持预读缓冲 | 无缓冲，每次指定 offset |
| 主要用途 | WAL 恢复、MANIFEST 回放 | SSTable 查询读取 |
| 性能优化 | 大块顺序 batch 读 | 按 Block 粒度随机读 |
| 缓存策略 | 预读 + 流式 | 块缓存 (BlockCache) |

### 2.5 文件类型 → 文件接口 映射关系

```
                        FileType
                           │
         ┌─────────────────┼──────────────────┐
         ▼                 ▼                   ▼
   kTableFile         kWalFile           kDescriptorFile
   kTempFile           ...                kCurrentFile
         │                 │                   │
         ▼                 ▼                   ▼
  RandomAccessFile   SequentialFile      SequentialFile
  (查询读)           (恢复读)            (回放)
  WritableFile       WritableFile        WritableFile
  (Compaction写)     (实时追加)          (变更追加)
```

### 2.6 Env 插件拦截点全景

将上述接口组合起来，完整的拦截链路覆盖了 RocksDB 的写入、读取、删除全生命周期。每个拦截点对应 `EnvWrapper` 中一个虚函数的重写，RocksDB 内核代码**零修改**。

---

#### 2.6.1 写入路径：SSTable（Compaction / Flush）

```
触发条件
  │
  ├─ Flush：MemTable 写满 → 冻结 → 刷盘生成 L0 SSTable
  │    触发点：DBImpl::FlushMemTableToOutputFile()
  │
  └─ Compaction：L_n 文件合并 → 生成 L_n+1 SSTable
       触发点：DBImpl::BackgroundCompaction()
              
拦截链路

  Step ①  Env::NewWritableFile("xxx.sst")          ← 拦截点
           └─ RemoteStorageEnv 接管
                ├─ 生成全局唯一 OBS 路径（shard_id + ts + seq）
                ├─ 创建 RemoteWritableFile（持有 obs_client_）
                └─ 返回给 RocksDB（零感知）

  Step ②  WritableFile::Append(data)                ← 拦截点
           └─ RemoteWritableFile::Append()
                ├─ 写入本地内存缓冲区（batch 聚合）
                ├─ 可选：写入时同步计算 checksum
                └─ 缓冲区满 → 触发异步上传（流水线）

  Step ③  WritableFile::Sync() / Close()            ← 拦截点
           └─ RemoteWritableFile::Close()
                ├─ 刷空缓冲区 → 完整上传到 OBS
                ├─ gRPC 注册到元数据中心
                │   { file_name, obs_path, size, shard_id, seq }
                ├─ 清理本地临时文件
                └─ 返回 OK → RocksDB 标记 Compaction 完成

数据流示意

  MemTable / Compaction Input
        │
        ▼
  RemoteWritableFile::Append()  ─── 缓冲区 ───→  异步上传 OBS
        │                                              │
        ▼                                              ▼
  RemoteWritableFile::Close()  ─── gRPC ──────→  Etcd 元数据注册
                                                        │
                                                        ▼
                                               MANIFEST 记录新文件
```

| 拦截点 | 虚函数 | 自定义行为 |
|--------|--------|-----------|
| ① 创建 | `Env::NewWritableFile` | 分配远端路径、创建 RemoteWritableFile |
| ② 追加 | `WritableFile::Append` | 缓冲聚合 + 异步上传流水线 |
| ③ 关闭 | `WritableFile::Close` | 完整上传 + 元数据注册 |

---

#### 2.6.2 写入路径：WAL（预写日志）

```
触发条件：每次写操作（Put / Delete / WriteBatch）
          触发点：DBImpl::WriteToWAL()

拦截链路

  Step ④  Env::NewWritableFile("xxx.log")           ← 拦截点
           └─ RemoteStorageEnv 接管
                ├─ 文件不存在 → 新建
                │   创建 DualWritableFile（local_fs_ + obs_client_）
                └─ 文件已存在 → 追加（ReuseWritableFile）

  Step ⑤  WritableFile::Append(WriteBatch)           ← 拦截点
           └─ DualWritableFile::Append()
                ├─ 序列化 WriteBatch → 二进制日志记录
                ├─ 写入本地文件（直接落盘，低延迟）
                └─ 写入远程缓冲区（异步批量上传）

  Step ⑥  WritableFile::Sync()                       ← 拦截点
           └─ DualWritableFile::Sync()
                ├─ ① 远程同步：等待远程缓冲区上传完成 + fsync
                │   └─ 远程成功 → 继续
                │   └─ 远程失败 → 返回错误
                ├─ ② 本地同步：local_fs_->Sync()（异步 fallback）
                └─ ③ 返回 OK → 写操作成功返回客户端

双写一致性保证

  客户端请求          KeyDB               WAL 引擎                远程 OBS          本地磁盘
     │                 │                   │                       │                │
     ├─ SET key ──────→│                   │                       │                │
     │                 ├─ WAL::Append() ──→│                       │                │
     │                 │                   ├── Append(local) ──────│───────────────→│  (异步，低优)
     │                 │                   ├── Append(remote) ────→│  (异步，高优)    │
     │                 ├─ WAL::Sync() ────→│                       │                │
     │                 │                   ├── Sync(remote) ──────→│  fsync           │
     │                 │                   │                    ←──│  OK              │
     │                 │                   ├── Sync(local) ────────│───────────────→│  fsync
     │                 │                   │                    ←──│────────────────│  OK
     │                 │←──── 返回 OK ─────│                       │                │
     ├─ 响应客户端 ←───│                   │                       │                │
```

| 拦截点 | 虚函数 | 自定义行为 |
|--------|--------|-----------|
| ④ 创建/复用 | `Env::NewWritableFile` | 创建 DualWritableFile（双写通道） |
| ⑤ 追加 | `WritableFile::Append` | 本地直接写 + 远程异步 buffer |
| ⑥ 同步 | `WritableFile::Sync` | 远程强同步 + 本地弱同步 |

---

#### 2.6.3 写入路径：MANIFEST（元数据变更）

```
触发条件：每次版本编辑
  - 新增 SSTable 文件（Compaction/Flush 完成）
  - 删除 SSTable 文件（Compaction 清理）
  - 列族创建/删除/配置变更
  触发点：VersionSet::LogAndApply()

拦截链路

  Step ⑦  Env::NewWritableFile("MANIFEST-xxx")      ← 拦截点
           └─ RemoteStorageEnv 接管
                ├─ 生成 etcd key: /manifest/{db_id}/{seq}
                ├─ 创建 ManifestWritableFile（meta_client_）
                └─ 本地不保留文件句柄

  Step ⑧  WritableFile::Append(VersionEdit)          ← 拦截点
           └─ ManifestWritableFile::Append()
                ├─ 序列化 VersionEdit → protobuf
                └─ 暂存到内存 buffer（批量提交）

  Step ⑨  WritableFile::Sync() / Close()             ← 拦截点
           └─ ManifestWritableFile::Sync()
                ├─ 事务提交到 Etcd
                │   etcd txn:
                │     cmp: 当前 version = expected
                │     then: put /manifest/{db_id}/{seq} → pb_data
                ├─ 更新 CURRENT 指针
                │   put /current/{db_id} → {latest_seq}
                └─ 返回 OK

版本管理示意

  MANIFEST-001  ──→  etcd key: /manifest/db1/001
  MANIFEST-002  ──→  etcd key: /manifest/db1/002
  MANIFEST-003  ──→  etcd key: /manifest/db1/003
                       ...
  CURRENT       ──→  etcd key: /current/db1  →  "003"
```

| 拦截点 | 虚函数 | 自定义行为 |
|--------|--------|-----------|
| ⑦ 创建 | `Env::NewWritableFile` | 创建 ManifestWritableFile ➔ Etcd |
| ⑧ 追加 | `WritableFile::Append` | 序列化 VersionEdit → buffer |
| ⑨ 同步/关闭 | `WritableFile::Sync` | Etcd 事务提交 + CURRENT 更新 |

---

#### 2.6.4 读取路径：SSTable 查询

```
触发条件：Get / Iterator 读取数据
  触发点：BlockBasedTable::Get() / BlockBasedTable::NewIterator()

拦截链路

  Step ⑩  Env::NewRandomAccessFile("xxx.sst")       ← 拦截点
           └─ RemoteStorageEnv 接管
                ├─ 检查本地缓存（LRU Cache）
                │   ├─ 缓存命中 → 使用本地缓存文件
                │   └─ 缓存缺失 → 创建 RemoteRandomAccessFile
                │       ├─ gRPC 查询元数据中心
                │       │   QueryLocation("xxx.sst") → {obs_path, size}
                │       └─ 返回 RemoteRandomAccessFile（持有 obs_client_ + cache_）
                └─ 返回给 RocksDB

  Step ⑪  RandomAccessFile::Read(offset, len)        ← 拦截点
           └─ RemoteRandomAccessFile::Read()
                ├─ 计算目标 Block 范围
                ├─ 查本地 BlockCache
                │   ├─ 命中 → 直接从缓存 buffer 返回
                │   └─ 缺失 → 从 OBS 范围读取
                │       ├─ obs_client_->Read(obs_path, offset, len)
                │       ├─ 存入本地 BlockCache
                │       └─ 返回数据
                └─ 更新缓存策略（LRU / 预读窗口）

缓存分层

   ┌──────────────────────────────────────────────┐
   │    Level 1: BlockCache（内存，Block 粒度）     │
   │    block_cache_->Lookup(block_offset)         │
   │    命中率 ≈ 90%+（业务热 Key 场景）            │
   └──────────────────┬───────────────────────────┘
                      │ miss
   ┌──────────────────▼───────────────────────────┐
   │    Level 2: 文件级本地缓存（SSD，文件粒度）    │
   │    local_cache_->Exists(file_name)           │
   │    缓存整个 SSTable 文件到本地磁盘            │
   └──────────────────┬───────────────────────────┘
                      │ miss
   ┌──────────────────▼───────────────────────────┐
   │    Level 3: 远程 OBS 存储                     │
   │    obs_client_->Read(path, offset, len)      │
   │    按 Block 范围读取，不缓存整个文件            │
   └──────────────────────────────────────────────┘

  预读优化：检测到连续读取 → 提前异步拉取后续 Block。
```

| 拦截点 | 虚函数 | 自定义行为 |
|--------|--------|-----------|
| ⑩ 打开 | `Env::NewRandomAccessFile` | 缓存决策 + 元数据查询 |
| ⑪ 读取 | `RandomAccessFile::Read` | 三级缓存查找 + 远端范围读 |

---

#### 2.6.5 读取路径：WAL 恢复

```
触发条件：DB 启动恢复 / 主备切换 Recover
  触发点：DBImpl::RecoverLogFiles()

拦截链路

  Step ⑫  Env::NewSequentialFile("xxx.log")         ← 拦截点
           └─ RemoteStorageEnv 接管
                ├─ ① 优先查询远程：从 Etcd 获取 WAL 列表
                │   └─ 最近 WAL 在远端 → 创建 RemoteSequentialFile
                ├─ ② 本地作为兜底：检查本地文件
                │   └─ 本地完整 → 使用本地 SequentialFile
                └─ ③ 远端 + 本地合并：WAL 可能被拆成多个片段

  Step ⑬  SequentialFile::Read(n)                    ← 拦截点
           └─ RemoteSequentialFile::Read()
                ├─ 从远端 OBS 流式读取 WAL 段
                │   以 Record 为单位拉取（非整个文件）
                ├─ 解包 Record（CRC 校验）
                │   └─ 校验失败 → 尝试本地片段补齐
                ├─ 组装成完整 WriteBatch
                └─ 返回给 RocksDB 回放

  Step ⑭  WAL 清理（恢复完成后）
           └─ 远程 WAL 标记为"已恢复"
               etcd txn: 更新 WAL 状态 → recovered

恢复路径选择

  远程 WAL 存在且完整？  ──Yes──→  从远端流式恢复（最近 N 个文件）
        │
        No
        │
  本地 WAL 存在？  ──────Yes──→  从本地恢复（fallback）
        │
        No
        │
  从远端全量拉取 + replay
```

| 拦截点 | 虚函数 | 自定义行为 |
|--------|--------|-----------|
| ⑫ 打开 | `Env::NewSequentialFile` | 远程优先 + 本地兜底策略 |
| ⑬ 读取 | `SequentialFile::Read` | 远端流式读 + CRC 校验 + 片段组装 |
| ⑭ 清理 | `Env::DeleteFile` | 远程状态标记 |

---

#### 2.6.6 删除路径：Compaction 文件清理

```
触发条件：Compaction 完成后，输入文件不再被引用
  触发点：VersionSet::ProcessCommitted() → 删除旧 SSTable

拦截链路

  Step ⑮  Env::DeleteFile("xxx.sst")                ← 拦截点
           └─ RemoteStorageEnv 接管
                ├─ ① 元数据中心标记删除
                │   meta_client_->DeleteFileRecord("xxx.sst")
                │   ├─ 逻辑删除（软删）：标记状态 = tombstone
                │   └─ 保留一段时间（GC 兜底）
                ├─ ② 远程存储删除
                │   if obs_client_->Exists("xxx.sst"):
                │       obs_client_->Delete("xxx.sst")
                │       └─ 异步删除（不阻塞主流程）
                ├─ ③ 本地缓存清理
                │   local_cache_->Evict("xxx.sst")
                │   local_fs_->Delete("xxx.sst")
                └─ ④ 返回 OK

  Step ⑯  延时 GC（后台线程，可选）
           └─ 扫描 Etcd 中状态 = tombstone 超过 TTL 的记录
               ├─ 物理删除 OBS 文件（批量）
               └─ 清除 Etcd 记录
```

| 拦截点 | 虚函数 | 自定义行为 |
|--------|--------|-----------|
| ⑮ 删除 | `Env::DeleteFile` | 软删 Etcd + 异步删 OBS + 清理本地缓存 |
| ⑯ GC | 后台定时任务 | 兜底物理删除（防残留） |

---

## 三、RocksDB Env 插件

### 3.1 Env 是什么

RocksDB 的抽象文件系统接口。默认实现是本地文件系统（`Env::Default()`）。通过继承 `EnvWrapper` 重写文件操作方法，实现存储重定向。

### 3.2 核心文件操作拦截

```cpp
class RemoteStorageEnv : public rocksdb::EnvWrapper {
public:
    // 创建文件（写入 SSTable 时调用）
    Status NewWritableFile(const std::string& fname,
                           std::unique_ptr<WritableFile>* result,
                           const EnvOptions& options) override {
        switch (GetFileType(fname)) {
            case FileType::SSTable:
                // SSTable → 写远程 OBS
                result->reset(new RemoteWritableFile(fname, obs_client_));
                break;
            case FileType::WAL:
                // WAL → 本地+远程双写
                result->reset(new DualWritableFile(fname, local_fs_, obs_client_));
                break;
            case FileType::MANIFEST:
                // MANIFEST → 元数据中心
                result->reset(new ManifestWritableFile(fname, meta_client_));
                break;
            default:
                return EnvWrapper::NewWritableFile(fname, result, options);
        }
    }

    // 随机读取（查询时读取 SSTable block）
    Status NewRandomAccessFile(const std::string& fname,
                               std::unique_ptr<RandomAccessFile>* result,
                               const EnvOptions& options) override {
        if (local_cache_->Exists(fname)) {
            return EnvWrapper::NewRandomAccessFile(fname, result, options);
        }
        result->reset(new RemoteRandomAccessFile(fname, obs_client_, local_cache_));
    }

    // 删除文件
    Status DeleteFile(const std::string& fname) override {
        meta_client_->DeleteFileRecord(fname);
        if (obs_client_->Exists(fname)) obs_client_->Delete(fname);
        local_fs_->Delete(fname);
    }
};
```

### 3.3 文件类型与存储位置

| 文件类型 | 后缀 | 作用 | 存储位置 |
|---------|------|------|---------|
| **SSTable** | .ldb/.sst | 实际数据文件 | OBS 远程存储 |
| **WAL** | .log | 预写日志 | 本地+远程双写 |
| **MANIFEST** | .manifest | 元数据清单 | 元数据中心 Etcd |
| **CURRENT** | CURRENT | MANIFEST 指针 | 元数据中心 |
| **OPTIONS** | .options | 配置 | 元数据中心 |
| **LOCK** | LOCK | 进程锁 | 本地 |
| **IDENTITY** | IDENTITY | 实例 ID | 本地 |

## 四、元数据中心

### 4.1 架构选型分析

元数据中心的本质是一个 **文件目录服务**：记录每个 SSTable / WAL / MANIFEST 文件的名称、存储位置、大小、状态、分片归属等信息。它不需要强一致的事务处理，但需要 **高可用、高吞吐、支持复杂查询**（按文件名查、按目录前缀枚举、按状态过滤等）。

业界方案主要有两类：

| 维度 | Etcd | MySQL 集群（+ gRPC 服务层） |
|------|------|---------------------------|
| **存储模型** | KV（层级 key） | 关系表（结构化字段 + 索引） |
| **查询能力** | 仅前缀/范围扫描 | 完整 SQL：WHERE + 多索引 + 排序 + 聚合 |
| **索引** | 仅有 key 前缀索引 | 二级索引、复合索引、全文索引 |
| **强一致性** | ✅ Raft 原生支持 | ✅ 事务（InnoDB redo log + MVCC） |
| **写入吞吐** | 受 Raft 提案速率限制（约 1000~2000 ops/s 单集群） | 高，MySQL 集群 / TiDB 可水平扩展 |
| **数据容量** | < 8GB 推荐，超 10GB 性能下降显著 | TB 级，无压力 |
| **单条大小** | 建议 < 1MB（Raft 提案传播代价高） | 无限制（BLOB 类型） |
| **Watch/订阅** | ✅ 原生 Watch 机制 | ❌ 需要轮询或 CDC（canal/debezium） |
| **锁服务** | ✅ Lease + Txn 原生支持 | ❌ 需额外实现（悲观锁/乐观锁） |
| **部署复杂度** | 简单，3 节点即可 | 较高，需 Proxy / 分库分表组件 |
| **运维成本** | 低 | 中等（连接池、慢查询、主从切换） |
| **典型定位** | 协调层：锁、选主、服务发现、小元数据 | 数据层：大规模结构化元数据存储 |

#### 场景匹配分析

| 元数据中心核心操作 | 适合 Etcd | 适合 MySQL | 原因 |
|-------------------|-----------|-----------|------|
| 文件注册（Insert） | △ | ✅ | 高频写，MySQL 吞吐更高 |
| 位置查询（Select by file_name） | △ | ✅ | 二级索引查，MySQL 更灵活 |
| 目录枚举（List by prefix） | ✅ | ✅ | Etcd 前缀扫描天然适合，MySQL 复合索引也可 |
| MANIFEST 版本管理 | ✅ | △ | 版本少、强一致要求高，Etcd 的 txn + watch 简洁 |
| 分布式锁 | ✅ | ❌ | Etcd lease + txn 原生支持 |
| 大容量历史记录 | ❌ | ✅ | 文件数量千万级时 Etcd 性能坍缩 |
| 复杂统计（group by shard） | ❌ | ✅ | SQL 聚合查询 |

**结论：元数据中心用 MySQL 集群 + gRPC 服务层更合理；Etcd 退回到协调层（锁、选主、配置下发）。**

```
┌─────────────────────────────────────────────────────┐
│              元数据中心架构（推荐）                    │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │           Go 元数据服务 (gRPC Server)          │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐  │ │
│  │  │ 文件注册  │ │ 位置查询  │ │ MANIFEST管理 │  │ │
│  │  └──────────┘ └──────────┘ └──────────────┘  │ │
│  │              连接池 + 读写分离                   │ │
│  └──────────────────────┬─────────────────────────┘ │
│                         │                            │
│  ┌──────────────────────▼─────────────────────────┐ │
│  │              MySQL 集群（主从 / TiDB）           │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐   │ │
│  │  │file_meta │ │ manifest │ │   cluster    │   │ │
│  │  │(文件元数)│ │ (清单)   │ │ (集群元数据)  │   │ │
│  │  └──────────┘ └──────────┘ └──────────────┘   │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│              ┌──────────────────────────┐            │
│              │  Etcd（协调层，轻量）      │            │
│              │  - 分布式锁              │            │
│              │  - Leader 选主           │            │
│              │  - 配置下发              │            │
│              └──────────────────────────┘            │
└─────────────────────────────────────────────────────┘
```

### 4.2 MySQL 元数据表设计

```sql
-- 文件元数据表
CREATE TABLE file_meta (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_name    VARCHAR(256) NOT NULL,           -- 原始文件名，如 000123.sst
    obs_path     VARCHAR(1024) NOT NULL,          -- OBS 存储路径
    file_type    TINYINT NOT NULL,                -- 1=SSTable 2=WAL 3=MANIFEST
    shard_id     INT NOT NULL,                    -- 分片 ID
    file_size    BIGINT NOT NULL DEFAULT 0,
    seq_num      BIGINT NOT NULL,                 -- 版本序列号
    status       TINYINT NOT NULL DEFAULT 0,      -- 0=active 1=tombstone 2=recovered
    checksum     VARCHAR(64),
    created_at   DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    deleted_at   DATETIME(3) DEFAULT NULL,        -- 软删除时间

    UNIQUE KEY uk_file_name (file_name),
    KEY idx_shard_id (shard_id, status),
    KEY idx_obs_path (obs_path(128)),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- MANIFEST 版本表
CREATE TABLE manifest_version (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    db_id        INT NOT NULL,
    version_seq  BIGINT NOT NULL,
    manifest_data   JSON NOT NULL,               -- VersionEdit 序列化内容
    current_flag TINYINT NOT NULL DEFAULT 0,     -- 1=当前最新版本
    created_at   DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),

    UNIQUE KEY uk_db_version (db_id, version_seq),
    KEY idx_current (db_id, current_flag)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 集群拓扑表
CREATE TABLE cluster_nodes (
    node_id      INT AUTO_INCREMENT PRIMARY KEY,
    host         VARCHAR(128) NOT NULL,
    port         INT NOT NULL,
    role         TINYINT NOT NULL,               -- 1=master 2=replica
    shard_range  VARCHAR(64),                    -- hash 范围
    status       TINYINT NOT NULL DEFAULT 1,
    last_heartbeat DATETIME(3)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4.3 gRPC 接口定义

```protobuf
package metapb;

service MetaService {
    // 文件注册：Compaction/Flush 完成后注册新文件
    rpc RegisterFile(RegisterFileReq) returns (RegisterFileResp);

    // 位置查询：查询文件实际存储路径
    rpc QueryLocation(QueryLocationReq) returns (QueryLocationResp);

    // 批量列举：按分片/状态枚举文件
    rpc ListFiles(ListFilesReq) returns (ListFilesResp);

    // 删除记录：软删除文件元数据
    rpc DeleteFileRecord(DeleteFileReq) returns (DeleteFileResp);

    // MANIFEST 相关
    rpc AppendManifest(AppendManifestReq) returns (AppendManifestResp);
    rpc GetCurrentManifest(GetCurrentManifestReq) returns (GetCurrentManifestResp);

    // 心跳 / 节点管理
    rpc Heartbeat(HeartbeatReq) returns (HeartbeatResp);
}

message FileInfo {
    string file_name   = 1;
    string obs_path    = 2;
    int32  file_type   = 3;
    int32  shard_id    = 4;
    int64  file_size   = 5;
    int64  seq_num     = 6;
    string checksum    = 7;
}

message QueryLocationReq  { string file_name = 1; }
message QueryLocationResp {
    string obs_path  = 1;
    int64  file_size = 2;
    int32  shard_id  = 3;
    bool   exists    = 4;
}
```

### 4.4 文件创建流程（MySQL 版）

```
RocksDB 创建 SSTable
  → Env::NewWritableFile()
    → 生成 OBS 路径（shard_id + timestamp + seq）
    → 创建 RemoteWritableFile（本地缓冲）
  → RemoteWritableFile::Append(data)
    → 写到内存 buffer
  → RemoteWritableFile::Close()
    ├─ 完整上传到 OBS
    ├─ gRPC → 元数据服务.RegisterFile()
    │   ├─ INSERT INTO file_meta(file_name, obs_path, file_type, shard_id, ...)
    │   ├─ 返回 file_id
    │   └─ 记录审计日志（可选）
    └─ 返回成功
```

### 4.5 文件读取流程（MySQL 版）

```
RocksDB 读取 SSTable
  → Env::NewRandomAccessFile()
    └─ 检查本地缓存
        ├─ 命中 → 直接打开本地文件
        └─ 缺失 → gRPC → 元数据服务.QueryLocation()
            ├─ SELECT obs_path, file_size FROM file_meta
            │  WHERE file_name = ? AND status = 0
            ├─ 返回 OBS 路径
            └─ 从 OBS 下载到本地缓存 → 打开
```

### 4.6 MANIFEST 管理（MySQL 版）

```
启动恢复流程

  Step 1  查询当前最新 MANIFEST 版本
           SELECT version_seq, manifest_data
           FROM manifest_version
           WHERE db_id = ? AND current_flag = 1

  Step 2  按序回放 VersionEdit
           manifest_data JSON 中按 seq 排列的编辑记录

  Step 3  启动完成后，恢复过程中的版本变更：
           INSERT INTO manifest_version(db_id, version_seq, manifest_data, current_flag)
           VALUES (?, ?, ?, 1)
           并 UPDATE 旧记录的 current_flag = 0

MANIFEST 变更流程（运行时）

  RocksDB 版本编辑
    → Env::NewWritableFile("MANIFEST-xxx")
      → gRPC → 元数据服务.AppendManifest()
        ├─ 事务 BEGIN
        ├─ INSERT INTO manifest_version(...)
        ├─ UPDATE current_flag = 0 WHERE db_id = ? AND current_flag = 1
        ├─ COMMIT
        └─ 返回新版 version_seq
```

### 4.7 一致性保障

元数据中心不同数据对一致性要求不同，可分层处理：

| 数据类别 | 一致性级别 | 策略 |
|---------|-----------|------|
| **MANIFEST 版本** | 强一致（线性） | 数据库事务 + 乐观锁版本号 |
| **文件注册** | 最终一致 | 主库写入，从库异步同步即可 |
| **位置查询** | 读已提交 | 允许短时间读到旧路径（文件已上传完成即可） |
| **删除标记** | 最终一致 | 软删除 + GC 兜底 |

### 4.8 高可用设计

```
                    ┌──────────────┐
                    │  Load Balancer│
                    │  (LVS/Nginx) │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │ Go MetaSvr1 │  │ Go MetaSvr2 │  │ Go MetaSvr3 │  ← 无状态，水平扩展
   └─────────────┘  └─────────────┘  └─────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │         MySQL 集群               │
          │  ┌──────┐ ┌──────┐ ┌──────┐   │
          │  │ M1   │ │ S1   │ │ S2   │   │  ← 主从 / MGR / TiDB
          │  │(主库) │ │(从库) │ │(从库) │   │
          │  └──────┘ └──────┘ └──────┘   │
          └────────────────────────────────┘
```

- **Go 元数据服务**：无状态，支持水平扩展，连接池复用 MySQL 连接
- **MySQL 集群**：主库处理 MANIFEST 强一致写，从库处理查询（读写分离）
- **缓存层**（可选）：file_meta 热点查询可加 Redis 缓存，降低 MySQL 读压力

## 五、深度问答

### Q1: Env 插件怎么拦截 RocksDB 文件操作？

> RocksDB 所有文件操作都通过 Env 接口。继承 EnvWrapper 重写 `NewWritableFile` / `NewSequentialFile` / `NewRandomAccessFile` / `DeleteFile` 等方法，RocksDB 内部不感知底层是本地还是远程存储。

### Q2: MANIFEST 文件怎么管理？

> MANIFEST 内容存到元数据中心（MySQL 或 Etcd），本地只保留缓存。启动时从元数据中心查询最新版本回放，变更时通过事务写入并更新 CURRENT 指针。

### Q3: 大 Key 分析怎么做？

> 计算内存占用时需遍历哈希桶，大量空桶消耗 CPU。优化：用 field 数量替代内存占用计算，减少遍历开销。

### Q4: 热 Key 分析增加多少内存？

> 每个 robj 增加 8 字节用于访问计数，总内存增加 < 5%，最多 < 10%。

# KeyDB 存算分离架构设计

> RocksDB Env 插件、存算分离、元数据中心、文件类型管理。

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
   │ (Etcd)  │                   │(OBS/S3) │
   └─────────┘                   └─────────┘
```

## 二、RocksDB Env 插件

### 2.1 Env 是什么

RocksDB 的抽象文件系统接口。默认实现是本地文件系统（`Env::Default()`）。通过继承 `EnvWrapper` 重写文件操作方法，实现存储重定向。

### 2.2 核心文件操作拦截

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

### 2.3 文件类型与存储位置

| 文件类型 | 后缀 | 作用 | 存储位置 |
|---------|------|------|---------|
| **SSTable** | .ldb/.sst | 实际数据文件 | OBS 远程存储 |
| **WAL** | .log | 预写日志 | 本地+远程双写 |
| **MANIFEST** | .manifest | 元数据清单 | 元数据中心 Etcd |
| **CURRENT** | CURRENT | MANIFEST 指针 | 元数据中心 |
| **OPTIONS** | .options | 配置 | 元数据中心 |
| **LOCK** | LOCK | 进程锁 | 本地 |
| **IDENTITY** | IDENTITY | 实例 ID | 本地 |

## 三、元数据中心（Etcd + gRPC）

### 3.1 gRPC 接口

```protobuf
service MetaService {
    rpc RegisterFile(FileInfo) returns (RegisterResponse);
    rpc QueryLocation(FileQuery) returns (FileLocation);
    rpc DeleteFileRecord(FileQuery) returns (DeleteResponse);
    rpc ListDir(DirQuery) returns (DirList);
}
```

### 3.2 文件创建流程

```
RocksDB 创建 SSTable
  → Env::NewWritableFile()
    → 生成 OBS 路径（shard_id + timestamp + seq）
    → 创建 RemoteWritableFile
  → RemoteWritableFile::Close()
    → 上传到 OBS
    → gRPC 注册到元数据中心
    → 返回成功
```

### 3.3 文件读取流程

```
RocksDB 读取 SSTable
  → Env::NewRandomAccessFile()
    → gRPC 查询元数据中心
    → 判断存储位置
      → LOCAL：直接读本地缓存
      → OBS：从 OBS 下载到本地缓存
    → 返回 RandomAccessFile
```

### 3.4 WAL 双写一致性

> 先写远程（同步）→ 再写本地（异步）→ 远程成功才返回。恢复时优先从远程拉取 WAL。

## 四、深度问答

### Q1: Env 插件怎么拦截 RocksDB 文件操作？

> RocksDB 所有文件操作都通过 Env 接口。继承 EnvWrapper 重写 `NewWritableFile` / `NewSequentialFile` / `NewRandomAccessFile` / `DeleteFile` 等方法，RocksDB 内部不感知底层是本地还是远程存储。

### Q2: MANIFEST 文件怎么管理？

> MANIFEST 内容存到元数据中心 Etcd，本地只保留缓存。启动时从元数据中心加载，变更时同步。

### Q3: 大 Key 分析怎么做？

> 计算内存占用时需遍历哈希桶，大量空桶消耗 CPU。优化：用 field 数量替代内存占用计算，减少遍历开销。

### Q4: 热 Key 分析增加多少内存？

> 每个 robj 增加 8 字节用于访问计数，总内存增加 < 5%，最多 < 10%。

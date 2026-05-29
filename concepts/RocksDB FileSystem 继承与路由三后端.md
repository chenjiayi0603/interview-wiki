# RocksDB FileSystem 继承与路由三后端

**简历原文**：继承 `rocksdb::FileSystem` 实现存算分离，按文件类型路由本地 SSD（热数据）/ 华为 OBS（冷数据）；元数据节点本地 TTL 缓存 + 熔断降级，节点故障不影响整体可用

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K1：`rocksdb--FileSystem`-继承-+-路由三后端.md]

## 数据支撑

| 后端 | 延迟 P50 / P99 | 路径 |
|------|----------------|------|
| 本地 SSD（热路径） | <0.1ms / <0.5ms | `kLocalPrefix` → posix 文件系统 |
| 元数据服务（gRPC） | 0.5–2ms | `kMetaPrefix` → gRPC 封装 |
| 华为 OBS（冷路径，长连接同 AZ） | 3–12ms / 20ms | `kObsPrefix` → OBS C++ SDK range get/put |
| OBS 含 TCP/TLS 建连（首包） | 8–25ms | 连接池复用后可降至长连接水平 |

热数据路径（热缓存命中率 95%+）整体 P99 <1ms。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K1：`rocksdb--FileSystem`-继承-+-路由三后端.md]

## 理论支撑

RocksDB 的所有文件 IO 最终经过 `rocksdb::FileSystem` 抽象层。继承此类，重写 `NewSequentialFile`（顺序读 SST）、`NewRandomAccessFile`（随机读 Block，读路径最频繁）、`NewWritableFile`（写 SST/WAL），即可替换整个 IO 路径而无需修改 RocksDB 上层逻辑。三个后端的错误码统一映射回 `rocksdb::IOStatus`，RocksDB 上层无感知。

RocksDB 原生设计就是为了换底层 IO：`rocksdb::Env::Default()` 本身即是 POSIX 实现，整个框架以 DI（依赖注入）方式传入 Env/FileSystem。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K1：`rocksdb--FileSystem`-继承-+-路由三后端.md]

## 业界对标

- **TiKV**：同样基于 RocksDB FileSystem 抽象扩展远程存储（TiFlash 存算分离）
- **Pika / SSDB**：Redis 协议 + RocksDB 底层，落盘路径类似但未做对象存储冷热分层
- **AWS Aurora + S3 存算分离**：计算与共享存储解耦的经典云原生模式

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K1：`rocksdb--FileSystem`-继承-+-路由三后端.md]

## 追问预案

**Q：三个后端的路由逻辑怎么实现？**
> 根据文件路径前缀分发：`/local/...` 走 SSD posix 实现；`/meta/...` 走 gRPC 封装；`/obs/...` 走 OBS SDK 的 range get/put。路由在 `FileSystem::NewXxxFile` 入口做一次字符串前缀匹配。

**Q：WAL 文件放在哪个后端？**
> WAL 和 L0 SST 文件必须留本地 SSD（`kLocalPrefix`），保证写入延迟 <1ms 且 Compaction 不直接打 OBS PUT 配额。只有 L1+ 的深层 SST 才路由到 `kObsPrefix`。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K1：`rocksdb--FileSystem`-继承-+-路由三后端.md]

## Related Pages
- [[KeyDB存算分离项目]]
- [[RocksDB文件体系]]
- [[RocksDB读取流程]]
- [[RocksDB性能分析与瓶颈]]
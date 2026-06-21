# KeyDB 存算分离 — 华为云 总览

> 基于 RocksDB Env 插件实现 Redis 存算分离，热内存 + 冷 OBS。

---

## 一、文件地图

| 文件 | 内容 |
|------|------|
| **01-KeyDB-架构设计.md** | 存算分离架构、RocksDB Env 插件、元数据中心 |
| **02-KeyDB-面试考点速查.md** | STAR 故事、性能数据、高频 Q&A |

## 二、项目速览

| 项目 | 数值 |
|------|------|
| 产品 | 华为云 KeyDB 存算分离存储引擎 |
| 技术栈 | C++ / RocksDB / Redis / OBS / Etcd / gRPC |
| 热数据 QPS | 150,000 读 / 80,000 写 |
| 冷数据延迟 | 8ms（需从 OBS 拉取） |
| 成本节省 | ~70% 对比纯内存方案 |

## 三、核心特性

- RocksDB Env 插件拦截文件操作，对上层无感
- 热数据本地 RocksDB Block Cache，冷数据 OBS 远程存储
- Etcd 元数据中心管理文件位置和 WAL/SSTable/MANIFEST
- 大 Key/热 Key 分析、布隆过滤器插件

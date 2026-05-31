# OBS 对接 RocksDB 性能分析

> 本文档对 KeyDB+RocksDB 对接 OBS 对象存储的性能预期进行深度验证与分析，涵盖延迟、QPS、Compaction 瓶颈、成本模型等核心维度。

See also: [[KeyDB存算分离项目]], [[RocksDB基础]], [[OBS对象存储]]

---

## 一、核心结论速查

| 指标 | KeyDB+RocksDB+OBS（最优配置） | 华为云 DCS 存储型 | 阿里云 Tair ESSD |
|------|-------------------------------|-------------------|------------------|
| **SET QPS** | **10~27 万** | ~7 万 | 8~9 万 |
| **热 GET QPS** | **20 万+**（内存级） | ~7 万 | 5~14 万 |
| **全冷 GET QPS** | **200~3,000** | ~7 万 | 2.5~5.1 万 |
| **半冷 GET QPS** | **11,000~33,000** | — | — |
| **SET 平均延迟** | **<0.1 ms** | 1~10 ms | 1~4 ms |
| **全冷 GET 串行延迟** | **60~180 ms** | 1~10 ms | 1~4 ms |

> **核心矛盾**：写性能可以碾压云厂商存储型，但全冷数据读性能差距 15~350 倍。universal compaction 用读放大换写放大，是双刃剑。

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 二、OBS 单次请求延迟分析

### 2.1 延迟构成

对象存储（OBS/S3）的一次 GET/PUT 请求延迟构成：

```
总延迟 = TCP连接建立 + TLS握手(HTTPS) + HTTP请求发送
       + OBS内部路由/鉴权 + 元数据查找
       + 数据定位(分片查找) + 磁盘IO
       + HTTP响应头部 + 数据传输 + TCP关闭
```

| 延迟构成 | 内网典型值 | 备注 |
|----------|-----------|------|
| 网络 RTT（同 Region 同 AZ） | 0.5~1 ms | 跨 AZ 1~3ms |
| TCP/TLS 建连（新连接） | 2~6 ms | 长连接/连接池复用可避免 |
| OBS 内部鉴权 & 路由 | 1~5 ms | 网关层处理 |
| 元数据查找 | 0.5~2 ms | KV 元数据引擎查询 |
| 数据定位 + 磁盘 IO | 0.5~3 ms | 取决于对象大小和冷热 |
| 数据传输（<1MB 小对象） | <1 ms | 内网带宽充裕 |
| **合计（热连接，小对象）** | **3~12 ms** | 同 AZ + 长连接 + 低负载 |
| **合计（含建连）** | **8~25 ms** | 含 TCP/TLS 建立 |
| **高负载/大对象场景** | **20~100+ ms** | 突发流量、跨 Region |

### 2.2 对 RocksDB 的含义

RocksDB 的 I/O 模式是**大量小块随机读写**：
- 每次 block cache miss 都是一次 OBS HTTP GET（4KB~64KB 小块）
- 每次 WAL 同步追加都是一次 OBS HTTP PUT（几 KB~几百 KB）
- 延迟由 OBS 的**处理开销**主导，而非带宽
- 不存在"摊薄"效应——小块 IO 无法像大对象那样用带宽换延迟

### 2.3 内网 SDK vs 公网 HTTP

| 指标 | 公网 HTTP REST | 内网 SDK |
|------|---------------|----------|
| 单次请求延迟（小对象 GET） | 5~50 ms | **1~5 ms** |
| 单次请求延迟（大对象 PUT） | 20~100 ms | **5~20 ms** |
| 读带宽（单连接） | 100~500 MB/s | **500~2,000 MB/s** |
| 写带宽（单连接） | 50~300 MB/s | **300~1,000 MB/s** |
| 多连接聚合带宽 | 受限于并发 | **可达 5~10 GB/s+** |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 三、Compaction 瓶颈分析

### 3.1 为什么 Compaction 是主要瓶颈

在 OBS 场景下，LSM-Tree 的 Compaction 写放大 10~30×（level compaction）全部变成远程 HTTP GET/PUT：

```
用户写入 1GB：
  Flush 到 L0:            1GB 写（OBS PUT × N 个 SST）
  L0→L1 Compaction:       读 L0 10 个 SST + 写 L1 SST  ≈ 1GB 读 + 1GB 写
  L1→L2 Compaction:       读 L1 10 个 SST + 写 L2 SST  ≈ 10GB 读 + 10GB 写
  L2→L3 Compaction:       ≈ 100GB 读 + 100GB 写
  ...逐层放大...

  不含 WAL 的写放大：10~30×
  含 WAL 后总写放大：11~31×
```

### 3.2 Write Stall 级联效应

在本地 NVMe 上，Compaction 通常几十~几百毫秒完成一次 L0→L1；在 OBS 上，每次 Compaction 需要**数秒~数十秒**。

RocksDB 的 write stall 机制：
1. L0 文件数超过 `level0_slowdown_writes_trigger` → 每次写入强制 sleep 1ms
2. L0 文件数超过 `level0_stop_writes_trigger` → **完全停止接受写入**
3. Immutable MemTable 积压超过 `max_write_buffer_number` → write stop

**OBS 上 stall 的触发频率和持续时间是本地 SSD 的 10~50 倍**，服务表现为间歇性不可用。

### 3.3 level vs universal compaction

| 维度 | level compaction | universal compaction |
|------|-----------------|---------------------|
| 写放大（不含 WAL） | 10~30× | 1.3~2.5× |
| OBS 总写入量（每 1GB 用户数据） | 11~31GB | 2.3~3.5GB |
| 读放大 | 低（每层最多 1 SST） | **高**（点查需遍历 10~30+ SST） |
| 冷 GET 的 OBS IO 次数 | 5~15 次 | **20~60 次** |
| 冷 GET 延迟（串行） | 30~100ms | **100~300ms** |
| 适合场景 | 读多写少 | 写多读少 |

> **核心矛盾**：universal compaction 用"读放大"换"写放大"。在 OBS 上两者都有严重问题——**不存在通过换策略就能"解决"的方案**。

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 四、SET QPS 分析

### 4.1 写路径延迟建模

```
SET(key, value)
  → 1. 写 WAL（sync fsync 到 OBS）：延迟 L_wal ≈ 5~30ms
  → 2. 写 MemTable（内存）：延迟 ≈ 0
  → 3. 返回客户端 OK

后台（不阻塞前台 SET）：
  → 4. MemTable 满 → Flush → 写 L0 SST 到 OBS：PUT ~100ms
  → 5. Compaction：读多条 SST + 写合并后 SST，累计数秒
```

### 4.2 WAL sync 策略对 QPS 的影响

| WAL 策略 | 持久性 | SET 延迟 | 串行 QPS 上限 |
|----------|--------|---------|-------------|
| `sync_log=true` → OBS | 强 | 5~30ms | 33~200 |
| `sync_log=false` → OBS（异步） | 弱 | ~0ms | 内存级 |
| `sync_log=true` → 本地 SSD | 强 | 0.1~0.5ms | 2,000~10,000 |

> **这是决定 SET QPS 的最核心变量**：sync_log=true 走 OBS 时，延迟直接由 OBS 决定。

### 4.3 最优配置下的 SET QPS 推导

**前提**：async WAL（`sync_log=false`，每秒 `FlushWAL()` 批量写）+ universal compaction + 内网 SDK

```
SET QPS = OBS_总带宽 ÷ (每 1MB 用户数据产生的 OBS I/O 量) ÷ avg_value_size

每 1MB 用户数据的 OBS I/O = 2 + 2×WA MB
  - WAL 批量 PUT: 1 MB
  - MemTable Flush: 1 MB
  - Compaction 读+写: WA × 2 MB

universal compaction (WA≈2): 每 1MB 用户数据 = 6 MB I/O
level compaction (WA≈20): 每 1MB 用户数据 = 42 MB I/O
```

| compaction 策略 | OBS 带宽场景 | 最大用户写入 MB/s | SET QPS @1KB |
|----------------|-------------|------------------|-------------|
| **universal** (WA≈2) | 中等 1,600MB/s | **267** | **26.7 万** |
| **level** (WA≈20) | 中等 1,600MB/s | **38** | **3.8 万** |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 五、GET QPS 分析

### 5.1 读路径与读放大

RocksDB 一次冷 GET 的物理 IO 分解（universal compaction，0% cache）：

```
GET(key) 执行流程：
  查 MemTable → 0 次 OBS IO（内存）
  查 Immutable MemTable(s) → 0 次 OBS IO（内存）
  
  遍历所有 SST（universal 下 10~30+ 个）:
    对每个 SST:
      ① 读 Filter Block  → 1 次 OBS GET
      ② 若 Bloom 判"可能存在":
         读 Index Block  → 1 次 OBS GET
         读 Data Block   → 1 次 OBS GET

总 OBS IO ≈ 20~60 次（universal compaction）
```

### 5.2 冷 GET QPS 三层分级

| 真实场景 | 定义 | OBS IO 次数 | 200 并发 QPS |
|----------|------|------------|------------|
| **全冷读** | block cache 0%，Index/Filter/Data 全部从 OBS 拉 | **20~60 次** | **1,000~3,300** |
| **半冷读** | Index/Filter cached，仅 Data Block 从 OBS 拉 | **1~3 次** | **11,100~33,300** |
| **热读** | block cache 100% | 0 次 | 内存级 20 万+ |

### 5.3 三组核心 QPS 汇总

| QPS 类型 | 值 | 核心公式 | 瓶颈 |
|----------|-----|---------|------|
| **热 GET** | **>20 万** | KeyDB 纯内存 benchmark | 内存带宽 / CPU 核心数 |
| **SET** | **10~27 万** | OBS带宽 ÷ (2+2×WA) ÷ value_size | OBS 写带宽 ÷ 写放大 |
| **半冷 GET** | **1.1~3.3 万** | 1000÷(1~3×3ms) × 200并发 × 效率 | OBS 单次延迟 × 1~3 次 IO |
| **全冷 GET** | **200~3,000** | 1000÷(20~60×3ms) × 200并发 × 效率 | **OBS 单次延迟 × 20~60 次 IO（读放大）** |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 六、OBS 服务端限流

| 限流维度 | 典型配额 | 影响 |
|----------|---------|------|
| 单桶 GET QPS | 5,000~20,000/s（标准存储） | GET 天花板 |
| 单桶 PUT QPS | 500~2,000/s（标准存储） | **Flush + Compaction 输出极易触顶** |
| 突发保护 | 超配额返回 503 SlowDown | 需客户端退避重试，进一步拉高 P99 |

> **单桶 PUT QPS 仅 500~2,000，而 Compaction 输出是大量 PUT 操作。即使客户端有无穷并发，PUT QPS 配额也会在 Flush/Compaction 高峰被迅速耗尽，触发 write stall。**

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 七、成本模型

### 7.1 Compaction 写放大导致的请求量爆炸

```
假设场景：业务 QPS = 2,000（SET+GET 混合），level compaction，写放大 15×（含 WAL）

业务请求/月：2,000 × 86,400 × 30 = 51.84 亿次
Compaction 额外请求 ≈ 51.84 亿 × (15 - 1) = 725.76 亿次
总 OBS 请求/月 ≈ 777.6 亿次

OBS 请求费（参考价 0.01 元/万次）：
  777.6 亿 / 10,000 × 0.01 = 7.78 万元/月

对比存储费（10TB × 0.1元/GB/月 = 1,000元/月）：
  请求费是存储费的 78 倍！

若改用 universal compaction（写放大 2.5×）：
  请求费 ≈ 1.3 万元/月（降至原来的 1/6）
```

> **即使性能上勉强接受，OBS 做主存储的请求费用也远超本地 SSD。Compaction 策略选择直接影响成本 6×。**

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 八、优化措施排名

| 排名 | 措施 | 预期效果 | 代价与风险 |
|------|------|---------|-----------|
| **1** | WAL 落本地 SSD，仅 SST 落 OBS | SET QPS 恢复至本地级 | 仍需本地 SSD，架构变复杂 |
| **2** | 增大 block_cache（可用内存 50~70%） | 读命中率↑，OBS GET↓ | 内存成本 |
| **3** | 采用 universal compaction | 写放大 10~30× → 1.3~2.5× | 冷数据 GET 读放大增加，延迟上升 |
| **4** | 增大 write_buffer_size（256MB~1GB） | 减少 flush 频率 → 减少 OBS PUT | 内存占用增加，崩溃恢复更久 |
| **5** | OBS 专用连接池 + 批量预取 | 提高并发效率 | 代码改造量大 |
| **6** | 冷热 SST tiering（热 SST 本地，冷 SST OBS） | 常见 compaction 走本地 | 需自定义 Env，工程量大 |
| **7** | 评估 RocksDB-Cloud 方案 | 原生支持 S3/OBS，有成熟 remote compaction | 依赖第三方项目，需评估稳定性 |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

---

## 九、最终选型建议

| 方案 | 可行性 | 推荐度 | 适用场景 |
|------|--------|--------|---------|
| OBS 做主存储（WAL+SST 全落） | 技术可行 | ❌ 不推荐 | 仅适合离线/归档/极低 QPS（<100） |
| OBS 做冷归档 | 简单可行 | ✅ 推荐 | 定期备份 SST，灾备恢复 |
| 混合架构（本地 SSD + OBS） | 技术复杂 | ⚠ 需评估 | 大容量冷热分层，有定制能力 |
| 使用云厂商托管存储型（华为云/阿里云） | 开箱即用 | ✅ 推荐 | 需要免运维、稳定性能 |

[src: raw/ingested/2技术/rocksdb/OBS对象存储对接RocksDB性能预期分析.md]

## Related Pages
- [[KeyDB存算分离项目]]
- [[RocksDB基础]]
- [[OBS对象存储]]
- [[Redis性能问题]]

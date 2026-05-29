# Compaction rate_limiter + WAL/L0 留本地 SSD

**简历原文**：Compaction `rate_limiter` 限速削峰，WAL/L0 留本地 SSD，规避 OBS 写入限流触发 write stall

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

## 数据支撑

| 维度 | 本地 SSD | 远程 OBS | 劣化倍数 |
|------|---------|---------|---------|
| 单次 IO 延迟 | 0.1–0.5ms | 5–30ms | 50–100× |
| Compaction P99 抖动 | 5–20ms | 50–200ms+ | 10× |
| OBS 单桶 PUT 配额 | — | 500–2,000/s | 硬天花板 |

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

Compaction 级联效应：PUT 打满 OBS 配额 → 503 SlowDown → 上传失败重试 → L0 文件堆积 → write slowdown → 极端情况 write stop → 业务写入暂停。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

缓解手段：
- Compaction `rate_limiter`：限速到 OBS 配额 70% 以内，大 Compaction 放在业务低峰期
- WAL + L0 留本地 SSD：仅 L1+ 深层 SST 落 OBS，Compaction 输出先写本地再异步上传
- 大 SST 文件（256MB）：减少 HTTP PUT 次数

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

## 理论支撑

LSM-Tree 写放大（Write Amplification Factor, WAF）：Compaction 过程将 L(N) 的 SST 与 L(N+1) 归并排序，每层数据在生命周期内会被写多次，WAF 约为 10–30×。在存算分离架构中，Compaction 输出直接打 OBS 是致命问题：每次 Compaction 产生的 SST 上传到 OBS 会消耗 PUT 配额，且 OBS 单桶配额是 500–2000/s 的硬上限。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

WAL 和 L0 留本地 SSD 的逻辑：WAL 是写入热路径，每次事务必须 fsync，本地 SSD fsync <0.5ms，OBS 延迟 5-30ms 无法承受。L0 文件未经 Compaction，写频率高，同理留本地。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

## 业界对标

- **RocksDB 官方文档** §Compaction：`rate_limiter` 是官方推荐的 Compaction 限速手段，`SetBytesPerSecond()` 动态调整
- **Snowflake / Databricks**：Compaction 卸载到独立计算集群（Compaction offload），在线节点只做读写，是更彻底的存算分离
- **TiKV 存储分离方案**：同样采用热数据本地 SSD + 冷数据对象存储，Compaction 处理方式类似

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

## 追问预案

**Q：write stall 和 write stop 有什么区别？**
> write slowdown 是 RocksDB 检测到 L0 文件数量超过阈值（`level0_slowdown_writes_trigger`，默认 20），主动降低写入速度（sleep 添加延迟）。write stop 是 L0 超过 `level0_stop_writes_trigger`（默认 36）时，业务写入完全暂停直到 Compaction 消化 L0。存算分离场景下 OBS 限流导致 Compaction 卡住，L0 堆积触发这两个阶段。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

**Q：rate_limiter 怎么设定具体数值？**
> 根据 OBS 单桶 PUT 配额（500–2000/s × 平均 SST 大小 256MB）计算 Compaction 输出最大带宽，限速在配额 70% 以内。高峰期进一步降低速率，低峰期（凌晨）放开跑大 Compaction。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K5：Compaction-rate_limiter-+-WAL-L0-留本地-SSD.md]

## Related Pages
- [[KeyDB存算分离项目]]
- [[OBS对接RocksDB性能分析]]
- [[RocksDB性能分析与瓶颈]]
- [[RocksDB线程配置最佳实践]]
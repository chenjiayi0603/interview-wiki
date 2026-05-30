# LRU 冷热分层（访问频次 × 时间衰减）

**简历原文**：按文件类型路由本地 SSD（热数据）/ 华为 OBS（冷数据）；LRU 缓存

## 数据支撑

| 指标 | 值 | 说明 |
|------|----|------|
| 热缓存命中率 | 95%+ | LRU 在业务热点局部性下的典型水平 |
| 冷请求量 | <5% | 低频路径，整体 P99 主要看热路径 |
| 冷数据 OBS P99 | ~20ms | 含内网 RTT + OBS 鉴权 + 数据 IO |
| 迁移粒度 | SST 文件级 | RocksDB 最小 IO 单元 = SST Block，文件级粒度与实际 IO 模式匹配 |

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K2：LRU-冷热分层（访问频次-×-时间衰减）.md]

## 理论支撑

打分公式：`score = access_count / (now - last_access_ts + 1)`，本质是单位时间访问密度。后台扫描线程每分钟跑一次，分数低于全部分数下四分位数的文件标记为"冷"，触发异步迁移。

与标准 LRU（单链表 O(1) 淘汰）的区别：标准 LRU 仅看最近访问时间，一次性线性扫描会错误 evict 真正的热数据（访问时间短但频次低的文件）。我们的方案兼顾频次，防止扫描风暴破坏真实热点。

与 OBS Lifecycle 策略的区别：OBS Lifecycle 是**时间维度**（N 天未操作 → 降级），我们是**热度维度**（访问密度打分）。纯靠 Lifecycle 节省约 30-40%，加上热度打分可到 70%+。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K2：LRU-冷热分层（访问频次-×-时间衰减）.md]

## 业界对标

- **Cassandra Tiered Storage**：SST 文件级冷热分层，与本设计同思路
- **HBase HFile 分层**：冷热数据分层到 HDFS，机制相同
- **RocksDB 自带 BlobDB**：大 value 独立存储 blob file，冷热分离的另一种形式

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K2：LRU-冷热分层（访问频次-×-时间衰减）.md]

## 追问预案

**Q：为什么不用 OBS Lifecycle 自动分层？**
> OBS Lifecycle 粒度是整对象 + 按时间，而我们需要 SST 文件 + 按访问热度。3 天前创建但业务热点一直读的文件，Lifecycle 会降级，我们 LRU 打分认为是热的。反过来今天才写但业务不再读的文件，Lifecycle 认为是标准存储，但应该迁到 OBS。纯靠 Lifecycle 节省率约 30-40%，加热度打分可到 70%+。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-K2：LRU-冷热分层（访问频次-×-时间衰减）.md]

## Related Pages
- [[KeyDB存算分离项目]]
- [[RocksDB文件体系]]
- [[OBS对接RocksDB性能分析]]
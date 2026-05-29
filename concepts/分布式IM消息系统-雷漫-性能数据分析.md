# 分布式 IM 消息系统 — 雷漫网络（性能数据分析）

> 任职：2019.11–2021.4 ｜ 角色：后台架构负责人
> 一句话：自研基于 Libev 的 C++ 分布式 IM 微服务，单机百万连接、万人群消息扇出从 3000 次压缩到 ~10 次。

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.1 速览卡

**性能**：
- 单机百万连接（调优后 8核16G，~8GB 内存）
- 消息端到端延迟 <50ms P99，可用性 99.99%
- 群消息 RPC：3000次 → ~10次（按网关节点合并）
- 在线状态查询：5–20ms → <1ms（一致性哈希 + LRU 缓存命中）
- msgseq 分配：99.99% 消息 <1ms（Redis INCR），批次预分配 5–20ms（每10000条触发一次）
- 未读数查询：<5ms/会话（MongoDB 联合索引 O(log M+K)）

**提升点**：
- 群消息扇出按 accnodeidentify 合并 RPC，从成员数量级压到网关节点数量级
- 一致性哈希路由 PushServer + 本地 LRU 缓存（TTL 5–30s），状态查询无需查 Redis
- msgseq Redis+MongoDB 分级预分配：正常路径不碰 MongoDB
- 未读数放弃 Redis INCR，改登录/断线重连查 MongoDB count（避免撤回DECR丢失/大群写扩散/故障清零）

**瓶颈**：
- 万人群带宽 3MB/条（全在线极端 10MB/条）
- MongoDB 单 Primary 写 QPS 上限 2–5万，超出需分片
- 长时间离线未读 K 值大，MongoDB count 耗时 >100ms
- seq 断号（服务重启批次浪费）触发客户端批量补拉，增加 MongoDB 读压力
- LRU 缓存一致性：群成员变更需订阅 RocketMQ 事件主动淘汰

**优化方向**：令牌桶限流 + MQ 削峰；超载降级只推@消息；登录时首屏懒加载前N会话；活跃群动态加大 seq 批次到50000；服务重启前写回最大已分配seq

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.2 核心性能指标表

| 指标 | 数值 | 业界对标 / 出处 |
|------|------|--------------|
| 机器配置 | **8 核 16G** | 美团 IM 网关、微信开放分享均在 8-16 核区间 |
| 承载连接数 | **1,000,000+** | WhatsApp 单机 2M 连接；C10M 命题 |
| 每连接内存 | **~8KB**（总 ~8GB/1M 连接） | tcp_rmem/wmem min 1KB + 内核元数据 ~4-5KB + 应用层 ~2-3KB/连接 |
| 调优前连接上限 | **约 30 万连接** | 默认 TCP buffer 下 16G 内存的物理上限 |
| CPU @ 空闲心跳 | **5–10%** | epoll ET 模式空闲连接近零唤醒成本 |
| CPU @ 业务高峰 | **30–50%** | Google SRE 容量规划标准 |
| 单跳内部转发延迟 | **<5ms** | gRPC / brpc 同 DC 公开 benchmark 1-3ms |
| **服务端处理 P99 延迟** | **<50ms** | 微信 P99 <100ms、钉钉 <80ms、飞书 <60ms（不含客户端网络 RTT）|
| 单机推送 QPS | **2–5 万 msg/s** | 美团 IM、知乎 IM 公开分享 |
| 心跳间隔 | **30–60s** | 微信智能心跳 4.5min；移动端 IM 业界 30s-5min |
| 群消息扇出比 | **3000 → 10** | 万人群在线率 ~30%，按网关聚合后等于网关节点数 |
| msgseq 分配 | **99.99% 消息 <1ms** | Redis Lua 原子 INCR；MongoDB 预分配 5-20ms/10000 条 |
| MongoDB count 延迟 | **<5ms**（正常），10–50ms（长离线 >10万未读） | 联合索引 `{sessionId, msgId}`，O(log M+K) |
| 在线状态查询 | 5–20ms → **<1ms** | 一致性哈希 + LRU 缓存（TTL 5-30s）+ RocketMQ 主动失效 |
| 关键内核参数 | ulimit -n=**1048576**, tcp_rmem/wmem min=**1KB**, conntrack_max=**2M** | 百万连接调优核心三项 + conntrack |
| 可用性 | **99.99%** | 年停机 <52 分钟 |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.3 单机百万连接实现

**调优四层结构（面试按这个顺序讲）：**

#### 第一层：内核参数

| 调优点 | 调优前 → 调优后 | 原理 |
|-------|---------------|------|
| **TCP 收发 buffer** | `tcp_rmem/wmem` min 4KB → **1KB** | 默认 1M 连接吃 100GB 内存；IM 多为小包 |
| **文件描述符** | `ulimit -n` 1024 → **1048576+** | 同步改 `fs.file-max`、systemd `LimitNOFILE` |
| **accept 队列** | `net.core.somaxconn` → **65535** | 配合 `tcp_max_syn_backlog` |
| **TIME_WAIT** | `tcp_max_tw_buckets` 调大 | 长连接场景影响小，弱网重连关键 |
| **conntrack 表** | `nf_conntrack_max` 调大 | 易被忽略，百万连接下不调会丢包 |

#### conntrack 深挖（最容易讲赢的细节）

Linux 内核的连接追踪子系统（netfilter），记录每个 TCP/UDP 连接状态。`iptables` 的 stateful firewall 和 NAT 都依赖它。

| 项 | 默认值 | 百万连接需求 |
|----|--------|-------------|
| `nf_conntrack_max` | 65,536 / 262,144（视发行版）| **2,097,152+** |
| 每 entry 内存 | ~300 字节 | 100 万 ≈ 300MB |

表满时内核直接丢 SYN 包，`dmesg` 刷：`nf_conntrack: table full, dropping packet`。

**生产标准配置（/etc/sysctl.d/99-conntrack.conf）：**
```
net.netfilter.nf_conntrack_max=2097152
net.netfilter.nf_conntrack_buckets=524288
net.netfilter.nf_conntrack_tcp_timeout_established=600
net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

**压测故事**：
> 压测到 50 万连接出现偶发 SYN drop，查 socket buffer 和 fd 都正常，最后 `dmesg` 看到 `nf_conntrack: table full`。调大 max 到 200 万、缩短 established 超时从 5 天到 10 分钟，后续压到 100 万，表使用率才 60%。

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

#### 第二层：网络模型

| 调优点 | 关键设计 |
|-------|---------|
| **epoll ET + 非阻塞** | while-EAGAIN 循环读，比 LT 减少 60%+ 系统调用 |
| **多进程 Worker + SO_REUSEPORT** | 内核负载均衡，避免 accept 锁竞争和惊群（参考 Nginx）|
| **零拷贝 / writev** | 多消息合并写，减少系统调用 |
| **心跳 30s + 滑窗判活** | 平衡功耗和断线感知 |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

#### 第三层：业务架构（核心亮点）

| 调优点 | 调优前 → 调优后 | 说明 |
|-------|---------------|------|
| **群消息扇出聚合** | **3000 次 RPC → ~10 次** | 按 `accnodeidentify(ip:port:workIndex)` 聚合 |
| **在线状态查询** | **5-20ms → <1ms** | 一致性哈希路由 PushServer + LRU 缓存 + RocketMQ 主动失效 |
| **msgseq 高并发分配** | 纯 Redis INCR 瓶颈 → **99.99% <1ms 不碰 MongoDB** | Redis Lua + MongoDB 预分配批次（每 1 万 FINDANDMODIFY）|
| **未读数方案** | 避免 Redis INCR/DECR 一系列问题 | 已读点存 msgId + 登录/重连查 MongoDB countDocuments |
| **消息可靠性** | 推送失败丢失 → **指数退避 + 离线兜底** | ZSet score=μs 时间戳，1s/2s/4s，超时转 APNs/FCM |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

#### 第四层：压测验证

| 工具 / 手段 | 用途 |
|------------|------|
| **自研压测客户端**（同套 Libev）| 单机模拟 50 万连接，多机叠加打满 1M |
| **tcpkali / wrk** | HTTP 网关辅助压测 |
| **火焰图**（perf + FlameGraph）| CPU 热点函数定位 |
| **`ss -s` / `nstat`** | 协议栈错误统计、连接状态分布 |
| **真实流量回放** | 凌晨低峰拷贝实流量到压测环境 |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.4 群消息扇出核心架构

**职责分离设计**：MsgServer 只做校验+落库+获取成员列表，通过 MQ 把任务交给 PushServer。在线状态查询和扇出在 PushServer 完成。

**完整流程：**
```
发送方 App → Nginx(SSL) → Access网关 → MsgServer
  ├─ 生成 msgId（Snowflake）
  ├─ 异步写 MongoDB（msg_c2g，分片键=groupId）
  ├─ 查 Redis 群成员缓存
  └─ 发布到 MQ（按 groupId 分区保序）
        ↓
PushServer（按 groupId 一致性哈希路由）
  ├─ 查群成员在线状态
  │    命中本地 LRU → <1ms
  │    未命中 → Redis pipeline HMGET → 5–20ms
  ├─ 按 accnodeidentify 分组聚合（同网关合并）
  ├─ 向各 Access 网关批量下发 cmd=4503
  │    Access 网关本地遍历连接表推送
  └─ 推送失败 → 指数退避重试(1s/2s/4s) → 超时转 APNs/FCM
```

**性能数字拆解：**

| 维度 | 数据 | 说明 |
|------|------|------|
| 万人群在线成员 | 约 30%，即 3000 人 | 典型 DAU/注册比 |
| 单条消息大小 | ~1KB | 含协议头、消息体、签名 |
| 单条群消息出口带宽 | 3000 × 1KB = **3MB/条** | 按在线 3000 人算 |
| 万人全在线极端情况 | 1万 × 1KB = **10MB/条** | 带宽爆炸临界点 |
| 网关节点分组后 RPC 次数 | 网关节点数（如 10 个） | 从 1万次降为 10 次 |

**为什么在线状态查询必须在 PushServer 而不在 MsgServer？**

| 对比点 | 在 MsgServer 查 | 在 PushServer 查 |
|--------|-----------------|------------------|
| 职责清晰度 | MsgServer 耦合推送拓扑 | 关注点分离，各司其职 |
| LRU 缓存有效性 | 无法利用 PushServer 本地 LRU | 一致性哈希保证同群命中同实例 |
| 扩展性 | MsgServer 扩容影响推送逻辑 | 独立扩缩容 |
| 在线状态时效 | 查完到推送有时间差 | 紧邻推送，时效更准 |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.5 msgseq 预分配机制

**核心设计：**
- 正常路径：Redis Lua 原子 INCR，<1ms，99.99% 消息走此路径
- 批次耗尽（每 10000 条）：MongoDB FINDANDMODIFY 预分配 5–20ms
- 冷启动（首条消息）：MongoDB INSERT 初始化 10–30ms

**seq 分配性能数字：**

| 路径 | 触发条件 | 延迟 | 频率 |
|------|----------|------|------|
| 正常分配（Redis INCR Lua） | 99.99% 的消息 | < 1ms | 每条消息 |
| 批次预分配（MongoDB FINDANDMODIFY） | 每 10000 条消息 1 次 | 5–20ms | 极低 |
| 冷启动（首条消息） | 会话第一条 | 10–30ms | 极低 |
| 并发竞争（SETNX 失败） | 多节点同时触发预分配 | 额外 1 次 Lua | 极低 |

**seq 断号问题与处理：**

| 场景 | 后果 | 处理 |
|------|------|------|
| 服务重启（预分配段未用完） | seq 浪费 0~10000 个，产生断号 | 客户端检测到 seq gap → 触发补拉请求 |
| 节点扩缩容 | MongoDB 换 nodeId 分配新段 | seq 整体单调，无影响 |
| 批量补拉 | seq gap 大时客户端补拉请求增加 | MongoDB 读压力瞬时增大 → 优化：活跃群动态加大批次到 50000 |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.6 未读数方案：为什么放弃 Redis INCR

| 方案 | 写入路径 | 撤回处理 | 故障恢复 | 大群影响 |
|------|---------|---------|---------|---------|
| **Redis INCR** | 每发一条消息，N 个成员各 INCR 1次 → N 次写 | 需 DECR，异步流程可能 DECR 丢失 | Redis 故障重连 → 计数器清零 → 未读数全丢失 | 大群 1 条消息 → N 次 Redis 写扩散 |
| **已读点 + count**（本项目） | 仅在登录/重连时查 MongoDB 一次 `countDocuments` | 统计 SQL 自然排除 `msgState=4`（撤回） | Redis 故障不影响（已读点丢失最多重算一次） | 大群无写扩散，仅 1 次 MongoDB count |

**为什么不用 Redis INCR 的详细分析：**

| 场景 | 为什么 Redis INCR 失效 | 具体后果 |
|------|----------------------|---------|
| 撤回消息（msgState=4） | INCR 写入容易，DECR 靠异步写回，服务挂掉时漏 DECR | 未读数永久偏高，红点不消 |
| 删除自己（msgState=5） | 只影响自己视图，客户端本地扣减；离线期间本地计数丢失 | 重新登录后无法从 Redis 还原准确值 |
| seq 断号 | 用 `max_seq - read_seq` 估算未读，断号号段无真实消息 | 结果偏大，红点虚高 |
| Redis 故障/重启 | Codis 重启后 INCR 计数器清零 | 消息还在 MongoDB，计数归零，未读全丢 |
| 大群写扩散 | 1 条消息 → N 个成员各 INCR 一次 | 万人群每条消息触发万次 Redis 写 |

**MongoDB countDocuments 性能数据：**

| 索引条件 | 延迟 | 说明 |
|---------|------|------|
| 联合索引 `{sessionId, msgId}` + `fromId` / `msgState` 过滤 | 正常 <5ms | 有索引覆盖或索引扫描 |
| 长时间离线（未读数 >10万条） | 10~50ms | K 值大但仍在 O(log M+K) 内 |
| 极端（半年未登录，百万级未读） | >100ms | 需引入快照辅助 |

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.7 消息可靠性保障

**推送状态 ZSet 生命周期：**
1. 消息产生 → ZADD key score=μs 时间戳 member=msgId
2. 推送到客户端 → PushServer 启动本地定时器（3s 后触发）
3. 正常路径：客户端回 ACK → ZREM → 定时器取消
4. 异常路径：3s 内未 ACK → 指数退避重试（1s→2s→4s）→ 超时 30s 转 APNs/FCM

**重发机制底层实现（Libev 定时器）：**
- `ev_timer`（一次性）：每次推送后注册，timeout=3s，检查是否 ACK
- `ev_periodic`（周期性）：全局 ZSet 扫描，interval=30s
- ZSet Key TTL=1 天：Redis 自身事件循环兜底清理，不触发重发

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.8 调优故事集

**故事 1：TCP buffer 调优**
- ❌ 问题：默认 `tcp_rmem/wmem` min=4KB → 百万连接内核 buffer ~8GB + 内核元数据 ~4GB + 应用 ~2GB = 14GB+，16G 机器扛不住。
- 🔧 手段：min 调到 1KB，内核 buffer → ~2GB。配合 conntrack 调优。
- 📊 效果：单机百万连接 ~8GB，CPU 空闲 5-10%。
- ⚠️ 代价：大包（>1KB）拥塞窗口偶发收缩，IM 99% 小包，能接受。

**故事 2：conntrack 表满（详见 §4.3 conntrack 深挖）**

**故事 3：群消息扇出聚合**
- ❌ 问题：万人群在线 ~3000 人 → 每条消息 3000 次 RPC → PushServer CPU 打满。
- 🔧 手段：按 `accnodeidentify` 分组合并，同网关节点成员合并成 1 次 RPC，网关本地 fanout。
- 📊 效果：推送从成员数量级压到网关节点数量级。
- ⚠️ 代价：网关需本地解包分发，CPU 多 5-10%。网关是 IO 密集型，余量充足。

**故事 4：msgseq 预分配**
- ❌ 问题：每次发消息 MongoDB FINDANDMODIFY（5-20ms）→ 消息路径不可用。纯 Redis INCR 重启丢 seq。
- 🔧 手段：Redis Lua 原子 INCR（<0.1ms）做主路径；额度耗尽时 MongoDB 预分配 10000 条（5-20ms）。99.99% 消息走 Redis。
- 📊 效果：消息路径延迟 5-20ms → <1ms，seq 会话内严格递增。
- ⚠️ 代价：重启浪费未用完段（最多 10000 seq 断号），客户端补拉瞬时增加 MongoDB 读压力。

**故事 5：未读数方案**
- ❌ 问题：Redis INCR 三个致命缺陷——大群写扩散、撤回 DECR 丢失、Redis 故障清零。
- 🔧 手段：已读点存 Redis `read_msgid`，登录/重连时 MongoDB count，在线客户端本地维护。
- 📊 效果：零写扩散，撤回自然排除，Redis 故障不影响。count <5ms。
- ⚠️ 代价：长离线（>10万未读）count 10-50ms，登录稍慢。离线时长远低于此，可接受。

**故事 6：消息可靠性**
- ❌ 问题：推送失败直接丢弃 → 用户收不到；重试风暴 → 打垮推送服务。
- 🔧 手段：ZSet 记录推送状态（score=μs），指数退避重试（1s→2s→4s），超时 30s 转 APNs/FCM，SETNX 防重发。
- 📊 效果：到达率 >99.9%，重试不风暴。
- ⚠️ 代价：ZSet 定期清理已 ACK 记录，每条消息额外一次 ZADD（微秒级可忽略）。

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.9 追问预案

**Q：百万连接 8GB 内存怎么算出来的？拆一下？**
> 内核层 ~4-5GB（TCP socket 元数据 3-4KB × 100万 + conntrack entry 300B × 100万）+ TCP buffer 层 ~2GB（tcp_rmem/wmem min=1KB, 100万×2KB）+ 用户态 ~2-3GB（Connection 对象 1-2KB/连接 + epoll 事件数组）= 8-10GB。实际压测 16G 机器跑到 100 万连接时 free 还能剩 6-7GB。

**Q：epoll ET vs LT 怎么选的？**
> 用 ET（边缘触发）。长连接场景下如果用 LT，只要 socket buffer 有数据，每次 epoll_wait 都返回该 fd——100 万连接哪怕只有 1% 有数据，也要处理 1 万个重复事件 CPU 空转。ET 的坑：必须 while-EAGAIN 读完，非阻塞 fd 是强制要求，EPOLLOUT 用完立即摘掉防惊群。

**Q：群消息扇出从 3000 压到 ~10 的具体流程？**
> MsgServer 只做校验+落库 → MQ → PushServer 消费 → 按 accnodeidentify 分组聚合 → 同网关合并 1 次 RPC → 网关本地 fanout。代价是增加一级 MQ 延迟（+5-10ms），但 RPC 从成员数量级压到网关节点数量级，PushServer CPU 大幅下降。

**Q：为什么放弃 Redis INCR 做未读数？**
> 三个致命缺陷：① 大群写扩散（万人群一条消息 1 万次 Redis 写）；② 撤回需要 DECR，异步流程下 DECR 可能丢失；③ Redis 故障计数器清零。我们选的是已读点 + MongoDB count，看起来"重"（登录时要 count），但在线时才最轻的，离线才查一次。

**Q：conntrack 表满怎么发现的？**
> 压测到 50 万连接出现间歇性 SYN 丢包。排查：socket buffer 正常 → fd 正常 → listen 队列正常 → `dmesg` 看到 `nf_conntrack: table full` → `/proc/sys/net/netfilter/nf_conntrack_count` 打满默认 65536。调大 max 到 200 万、缩短 ESTABLISHED 超时到 10 分钟，再压 100 万连接时表使用率 60%。

**Q：消息撤回怎么实现？**
> 撤回是发一条特殊的控制消息（type=revoke + 引用 msgId），客户端收到后本地 mark 那条消息为已撤回。服务端不删原消息（保留审计），只在拉取时打 revoke 标志。这也是为啥 Redis INCR 未读数会有问题——撤回 DECR 容易出错。

**Q：seq 断号怎么处理？**
> seq 断号是设计上知道会发生的 trade-off。好处是 99.99% 正常消息不碰 MongoDB。客户端检测到 seq gap 会触发补拉请求，活跃群动态加大批次到 50000。服务重启前写回最大已分配 seq 减少浪费。

**Q：百万连接数字是单机实测还是集群聚合？**
> 单机实测。8 核 16G 单台压测打到 100 万长连接，~8GB 内存。集群层面是水平扩展，多台叠加。

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

## 4.10 讲解话术

**30 秒版：**
> "雷漫 IM 系统，我是后台架构负责人。自研基于 Libev 的 C++ 异步框架，单机 8 核 16G 撑百万连接（内存 ~8GB）、服务端处理 P99 <50ms。亮点是群消息扇出优化：万人群推送从 3000 次 RPC 压到 ~10 次（按网关节点聚合），在线状态查询从 5-20ms 优化到 <1ms（一致性哈希 + LRU 缓存 + RocketMQ 失效），msgseq 设计让 99.99% 消息 <1ms 不碰 MongoDB。"

[src: raw/ingested/3项目/面试准备/0_面试项目性能数据分析-4.-分布式-IM-消息系统-—-雷漫网络-4.-分布式-IM-消息系统-—-雷漫网络.md]

---

## 补充：性能数字全解（来自面试准备文档）

[src: raw/ingested/3项目/面试准备/性能数据重点说明-四、分布式-IM-—-性能数字全解.md]

### 单机百万连接

**内存完整推算：**

```
每个 TCP 连接内存开销：
  内核态：
    tcp_rmem 最小值（调优后）：1KB
    tcp_wmem 最小值（调优后）：1KB
    socket 元数据结构：~1KB
    合计内核态：~3KB/连接
  用户态（应用层连接对象）：
    连接状态、UID、心跳时间戳等：~1–2KB/连接

百万连接总内存：
  内核态：3KB × 100万 = 3GB
  用户态：2KB × 100万 = 2GB
  合计：~5GB（调优后）
  默认配置（不调优）：~10GB+
```

**系统参数调优清单：**

| 参数 | 默认值 | 调优值 | 作用 |
|------|--------|--------|------|
| `ulimit -n` | 1024 | 1048576 | 文件描述符上限 |
| `fs.file-max` | 100000 | 1048576 | 系统级 fd 上限 |
| `net.ipv4.tcp_rmem` min | 4KB | 1KB | 接收缓冲最小值 |
| `net.ipv4.tcp_wmem` min | 4KB | 1KB | 发送缓冲最小值 |
| `net.core.somaxconn` | 128 | 65535 | accept 队列长度 |

**被问到时要补充：**
> "百万连接的核心挑战不是 epoll 的性能，而是内存。默认配置下每个连接约 10KB，百万连接就是 10GB。调优后 socket buffer 最小值从 4KB 降到 1KB，实际压测约 5GB 内存可以承载百万连接。压测用多台客户端机器，每台机器绑多个 IP 避免端口耗尽。"

[src: raw/ingested/3项目/面试准备/性能数据重点说明-四、分布式-IM-—-性能数字全解.md]

### 消息延迟 <50ms（P99）

**端到端链路拆解：**

```
发送端设备
  ↓ 移动网络/Wi-Fi（不可控，通常 10–50ms）
网关层（<1ms，本地处理）
  ↓ 内网（<0.5ms）
消息服务（<3ms，路由 + 权限校验）
  ↓ 内网（<0.5ms）
MongoDB 写入（<5ms，WriteConcern=1，Primary 写）
  ↓ 异步推送（不等待存储完成）
推送服务 → 接收端网关（<2ms）
  ↓ 长连接推送（<1ms）
接收端设备
```

**为什么能做到 P99 <50ms：**
- 关键优化：**先推送再落库**（异步写存储），推送路径不等 MongoDB 返回
- 代价：极端情况（MongoDB 写入失败）消息丢失，靠补偿机制（ACK + 重试）补救
- 测量环境：同机房内网测试，不含移动网络延迟

**被问到时要补充：**
> "这个 50ms 是服务端端到端的数字，测试环境在同机房内网。移动端用户实际体验会加上无线网络延迟，通常 50–200ms。服务端能控制的是尽量压缩，所以我们做了先推送再落库的优化，把存储 IO 从关键路径上摘除。"

[src: raw/ingested/3项目/面试准备/性能数据重点说明-四、分布式-IM-—-性能数字全解.md]

### 可用性 99.99%

**故障时间分配：**
- 年允许停机 52 分钟
- 单次发布滚动更新：约 30s（多实例轮换）
- 全年发布约 50 次：50 × 30s = 25 分钟
- 剩余故障预算：27 分钟/年，约每次故障 < 5 分钟

**保障手段：**
- Nacos 健康检查：实例故障 5–10s 内摘除
- 消息补偿：发送失败自动重试 3 次，间隔指数退避
- 多机房部署：跨机房流量切换（切换期间有短暂不可用）

[src: raw/ingested/3项目/面试准备/性能数据重点说明-四、分布式-IM-—-性能数字全解.md]
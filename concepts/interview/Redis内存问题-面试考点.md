# Redis 内存问题（面试考点）

See also: [[Redis内存问题]], [[Redis性能问题]], [[Redis可观测性项目]]

## 问题本质

Redis/KeyDB 集群突然内存涨了——可能是真的业务流量上来了，也可能是大 Key、热 Key、内存碎片、slowlog 积压、或某个 Client buffer 炸了。

## 面试答题框架：三步定位法

```
Step 1：是谁吃了内存？（快速定位元凶）
Step 2：为什么吃？（根因分析）
Step 3：怎么止血和根治？（短期+长期）
```

## 具体话术（口述版）

**"Redis 集群内存突然飙升，你怎么排查？"**

> 这个问题很典型，线上至少有五六种不同原因导致的内存涨。我按优先级说。
>
> **第一步，先用 `INFO memory` 看全局。**
> - `used_memory_rss` vs `used_memory`：如果差值大，说明是内存碎片
> - `mem_fragmentation_ratio` > 1.5 就要关注了
> - `maxmemory` 是否触发，是否在淘汰 key
>
> **第二步，用 `CLIENT LIST` 看客户端缓冲区。**
> - 很多内存突然涨是因为某个客户端订阅了大量 channel，或者 monitor 命令没关，output buffer 爆炸
> - 看 `omem`（输出缓冲区占用），如果某个 client 的 omem 很大，直接 `CLIENT KILL`
> - 保护措施：配置 `client-output-buffer-limit`，普通客户端 0（不限制）是有风险的
>
> **第三步，查大 Key。**
> - 用 `--bigkeys` 扫描，但不是实时的
> - 我们自研了大 Key 实时检测：hook 所有写命令，超过阈值就异步记录并告警
> - 大 Key 不只是内存问题，还会导致主从同步延迟、迁移失败、甚至主从切换时的全量同步 OOM
>
> **第四步，看慢日志和延迟。**
> - `SLOWLOG GET 100`，看有没有复杂度高的命令（KEYS *、HGETALL 大 hash、ZRANGE 大 set）
> - 如果有 DEL 一个大 key 被阻塞，可以用 `UNLINK` 异步删除
>
> **第五步，如果是集群模式，查热 Key 导致的流量倾斜。**
> - 某个 key 的 QPS 远超其他，导致对应分片内存和 CPU 都高
> - 我们做了热 Key 自动检测和本地缓存，检测到热 Key 后在 Proxy 层或 SDK 层做本地缓存，减轻后端压力
>
> **最终止血**：如果内存已经逼近上限，先扩容或切流，保证服务不 OOM；同时做 key 的 TTL 治理和冷数据下沉。

## 补充：内存碎片的处理

**Q: 内存碎片率很高怎么办？**

> Redis 内存碎片主要来自频繁的 key 增删，特别是大小不一的对象反复分配释放。处理方法：
> - Redis 4.0 后支持 `activedefrag`，可以在线整理碎片。但 CPU 消耗不小，高峰期慎开
> - 更根本的：减少频繁的 key 增删、用 jemalloc 替代 libc malloc、用 lazyfree 异步删除
> - 如果碎片率持续 > 2 且影响使用，重启是最快但最粗暴的方式，要做好主从切换再重启

## 更隐蔽的内存陷阱

| 陷阱 | 表现 | 排查方法 | 处理 |
|------|------|---------|------|
| **复制缓冲区积压** | 主从断连恢复期间，主节点的 replication buffer 暴涨 | `INFO replication` 看 backlog | 增大 `repl-backlog-size`，快速恢复主从 |
| **AOF 重写时 Copy-On-Write** | `bgrewriteaof` 期间内存翻倍 | 父子进程共享内存写时复制 | 避免在内存高水位时触发重写 |
| **Lua 脚本中创建大变量** | 脚本执行期间内存涨但不释放 | `SCRIPT KILL` | 脚本超时限制，禁止复杂操作 |
| **Pub/Sub 消息积压** | 消费者慢或离线，消息堆积在 output buffer | `CLIENT LIST` 看 subs 状态 | 限制 buffer 大小，必要时踢掉慢客户端 |
| **过期 key 未及时回收** | 大量 key 同时过期，但淘汰速度跟不上 | `INFO stats` 看 `expired_keys` | 随机化过期时间、开启 lazyfree-lazy-expire |

[src: raw/ingested/3项目/面试准备/生产环境故障排查-面试考点文档-4.-内存问题（面试核心-—-最体现功力）.md]
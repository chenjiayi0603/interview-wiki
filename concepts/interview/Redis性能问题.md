# Redis 性能问题（延迟高）

See also: [[Redis数据不一致]]

## 现象
- 响应慢
- 请求超时

## 可能原因
- 存在慢命令：大 key（如大 list/hash 全量操作）、`KEYS *`、复杂 Lua、大批量 HGETALL 等
- 内存不足触发 Swap，导致响应变慢
- 内存碎片率高，分配/回收开销大
- RDB 快照或 AOF 重写占用主线程（或 fork 后父进程仍受 copy-on-write 影响）
- 单实例 QPS 或数据量过大，未做分片或读写分离
- 网络带宽打满或延迟高（跨机房、跨可用区）
- 主从全量同步期间主节点负载升高

## 排查步骤
1. **基准延迟**：使用 `redis-cli --latency` 测量基准延迟
2. **慢查询日志**：通过 `SLOWLOG` 分析慢命令
3. **内存使用与碎片**：检查 `INFO memory` 中的碎片率
4. **持久化阻塞**：排查 RDB 快照、AOF 重写是否阻塞主线程
5. **Swap 使用**：检查是否发生内存 Swap

## 解决方案
- 优化慢查询与数据结构（如大 key、不合理的数据结构）
- 升级规格或集群分片
- 调整持久化策略（如改为异步持久化、调整触发条件）

## 根因原理深入分析

### 为什么单条慢命令会阻塞所有请求？
Redis 的核心命令处理是**单线程**的（即使 6.0 引入了多线程 I/O，命令执行仍是单线程）。这意味着：

```
客户端A请求 → [排队] → 执行KEYS * (耗时500ms) → 返回A
客户端B请求 → [排队等A完成] → 执行GET key → 返回B   ← B 被阻塞了500ms
```

**时间复杂度是关键**：
| 命令 | 时间复杂度 | 风险等级 |
|------|-----------|---------|
| `KEYS *` | O(N)，N=所有key数量 | 极高，生产环境禁止使用 |
| `HGETALL` 大hash | O(N)，N=field数量 | 高，field过多时应用HSCAN |
| `DEL` 大key | O(N)，N=元素数量 | 高，应使用UNLINK(异步删除) |
| `SORT` | O(N+M*log(M)) | 高 |
| `LRANGE 0 -1` 大list | O(N) | 高 |

**根因**：Redis 选择单线程模型是为了避免锁竞争和上下文切换的开销，在命令执行足够快（微秒级）时吞吐量极高。但这把双刃剑意味着任何一个 O(N) 的慢命令都会直接拖垮整个实例。

### fork + Copy-on-Write 导致延迟的原理
Redis 执行 `BGSAVE`（RDB快照）或 `BGREWRITEAOF` 时会 `fork()` 子进程：

```
fork() 前：
  父进程 → 页表指向物理内存页 [A][B][C][D]...

fork() 后：
  父进程 → 页表 → [A][B][C][D]... (共享，标记为只读)
  子进程 → 页表 → [A][B][C][D]... (共享，标记为只读)

父进程写入时触发 Copy-on-Write：
  父进程写页B → 内核复制页B为B' → 父进程指向B', 子进程仍指向B
```

**延迟产生的三个阶段**：
1. **fork 本身**：虽然不复制数据，但需要复制页表。10GB 数据约有 260 万个页表项（4KB/页），fork 耗时约 10-20ms。在此期间主线程完全阻塞
2. **CoW 期间**：每次父进程修改一个 page 都触发内核的 page fault，分配新物理页、复制 4KB 数据，耗时约几十微秒。写入密集时大量 page fault 会显著拖慢主进程
3. **内存翻倍风险**：极端情况下（写入覆盖所有页），内存占用翻倍，可能触发 OOM killer

**处理方式**：
```bash
# 关闭 THP（Transparent Huge Pages），避免 CoW 时复制 2MB 大页
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 监控 fork 耗时
redis-cli INFO stats | grep latest_fork_usec
# latest_fork_usec: 上一次 fork 耗时（微秒）

# 调整 RDB 触发策略，减少不必要的 fork
# redis.conf 中设置合理的 save 条件
```

### Swap 导致延迟的原理
当 Redis 进程使用的内存超出物理内存时，Linux 将部分内存页换出到磁盘（Swap）：
- 正常：内存访问延迟 ~100ns
- Swap：磁盘访问延迟 ~10ms（HDD）或 ~0.1ms（SSD）

**延迟放大 100-100000 倍**，且 Redis 每个命令可能访问多个内存页，一次 Swap page fault 就可能让微秒级命令变成毫秒级。

**处理方式**：
```bash
# 检查 Redis 进程是否使用了 Swap
cat /proc/$(pidof redis-server)/smaps | grep -i swap
# 如果 Swap 非 0，说明已经发生换页

# 建议：生产环境设置 vm.swappiness=1（不完全禁用，但最小化使用）
sysctl vm.swappiness=1
```

[src: raw/ingested/2技术/redis/Redis_KeyDB运维问题速查.md]

## Related Pages
- [[Redis数据不一致]]
- [[Redis内存问题]]
- [[Redis持久化问题]]
- [[Redis可观测性项目]]
- [[Redis连接问题]]
- [[性能优化]]

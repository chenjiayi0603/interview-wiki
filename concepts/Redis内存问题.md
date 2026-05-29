# Redis 内存问题（OOM）

See also: [[性能优化]]

## 现象
- OOM 错误
- 无法写入新数据

## 可能原因
- 未设置 `maxmemory` 或设置过大，超过物理内存
- 存在大 Key（大 string、大 list/hash/set），单 key 占用过多内存
- 键数量过多且无 TTL，冷数据长期占用
- `maxmemory-policy` 为 `noeviction`，写满后拒绝写入且未淘汰
- 内存碎片率高，`used_memory_rss` 远大于 `used_memory`，实际可用不足
- 持久化子进程 fork 导致内存翻倍（Linux 下 copy-on-write），触发 OOM

## 排查步骤
1. **内存使用率**：监控 `used_memory`、`maxmemory`
2. **大 Key 分析**：使用 `redis-cli --bigkeys` 或相关工具分析
3. **内存碎片率**：`mem_fragmentation_ratio` 是否异常
4. **淘汰策略配置**：检查 `maxmemory-policy` 是否合理

## 解决方案
- 清理大 Key 或拆分（如 list/hash 拆分）
- 设置合理淘汰策略（`allkeys-lru`、`volatile-lru` 等）
- 启用内存碎片整理：
  - 在 Redis 4.0 及以上，可以通过配置 `activedefrag yes` 启用主动碎片整理（建议结合 `active-defrag-threshold-lower/upper` 参数进行优化）。
  - 也可手动执行 `MEMORY PURGE` 命令（需要 Redis 4.0+），立即回收未使用的内存页。

**MEMORY PURGE 做了啥？**

`MEMORY PURGE` 是 Redis 4.0 及以上版本支持的命令，用于手动触发操作系统回收 Redis 进程占用但未实际使用的物理内存（释放“脏页”）。具体来说：

- **背景**：不少情况下，Redis 因频繁分配、释放大块内存（如删除大 Key、释放大对象），会导致进程虚拟内存下降但物理内存（RSS）未能即时释放，出现“used_memory 已减少但 RSS 仍很高”现象。这通常由于 Redis 使用的 jemalloc/malloc 分配器和操作系统的内存回收机制无法及时归还物理内存。
- **MEMORY PURGE 工作原理**：执行该命令后，Redis 会调用底层分配器（如 jemalloc 的 `malloc_trim`）主动通知操作系统尝试将空闲物理内存归还给操作系统，从而降低 `used_memory_rss`。
- **注意事项**：
  - `MEMORY PURGE` 不会影响数据安全，也不会清理 key，仅做内存回收。
  - 并非所有“内存碎片”都能被回收，能释放多少取决于底层 allocator 与内核支持（有时仅能回收部分）。
  - `MEMORY PURGE` 适合在出现内存释放后 RSS 居高不下的场景下手动执行。
  - 在 Redis 4.0+ 启用 `activedefrag` 后，碎片整理和 RSS 回收会更智能，日常手动执行的需求会减少。

**示例**：
```
127.0.0.1:6379> MEMORY PURGE
OK
```
执行后可通过 `INFO memory` 观察 `used_memory_rss` 是否下降。

## 根因原理深入分析

### 内存碎片产生的根本原因
Redis 使用 jemalloc 内存分配器，jemalloc 按照**固定大小分类**（size class）分配内存：8B, 16B, 32B, 48B, 64B, 80B, 96B...

```
请求 40 字节 → jemalloc 分配 48 字节 → 浪费 8 字节（内部碎片）

频繁创建/删除不同大小的对象 → 内存空间出现"空洞" → 外部碎片
  [占用][空闲32B][占用][空闲64B][占用][空闲16B]
  此时需要分配 100B，虽然总空闲 > 100B，但没有连续空间 → 必须申请新内存
```

**碎片率公式**：`mem_fragmentation_ratio = used_memory_rss / used_memory`
- `> 1.5`：碎片严重，大量内存被浪费
- `< 1`：说明发生了 Swap（used_memory 包含 Swap 部分），非常危险
- `1.0 ~ 1.5`：正常范围

**高碎片的典型场景**：
- 大量小 key 频繁过期和创建（如会话管理）
- 频繁修改 value 大小（如 string 从 100B 变为 1KB 再变为 50B）
- 大 key 删除后留下大块空洞

### 淘汰策略的工作原理
当 `used_memory` 达到 `maxmemory` 时，Redis 根据 `maxmemory-policy` 决定行为：

| 策略 | 范围 | 算法 | 适用场景 |
|------|------|------|---------|
| `noeviction` | - | 拒绝写入，返回错误 | 不允许数据丢失 |
| `allkeys-lru` | 所有key | 近似LRU | 通用缓存场景（最常用） |
| `volatile-lru` | 有TTL的key | 近似LRU | 缓存+持久数据混合 |
| `allkeys-lfu` | 所有key | 近似LFU | 访问频率差异大的场景 |
| `volatile-ttl` | 有TTL的key | 按TTL排序 | 优先淘汰即将过期的 |
| `allkeys-random` | 所有key | 随机 | 访问完全均匀 |

**近似 LRU 的实现原理**：Redis 并不维护真正的 LRU 链表（开销太大），而是采用**采样淘汰**：
1. 随机采样 `maxmemory-samples`（默认5）个 key
2. 比较它们的最后访问时间（`lru` 字段，24bit，精度约 10 秒）
3. 淘汰其中最久未访问的那个
4. 重复直到内存降到 `maxmemory` 以下

采样数越大越接近真实 LRU，但 CPU 开销越高。生产环境建议设为 10。

### fork 导致 OOM 的原理
`BGSAVE`/`BGREWRITEAOF` 的 fork 子进程与父进程共享内存，但如果父进程在子进程工作期间有大量写入，CoW 会导致物理内存使用接近翻倍：

```
Redis 使用 8GB 内存（maxmemory=8GB，物理内存 10GB）
执行 BGSAVE → fork 子进程
写入密集 → CoW 复制了 6GB → 总物理内存使用 8 + 6 = 14GB > 10GB
→ 触发 Linux OOM Killer，杀掉 Redis 进程
```

**处理方式**：
```bash
# 预留足够内存给 fork：建议物理内存 >= maxmemory * 2
# 或者调整 overcommit 策略
sysctl vm.overcommit_memory=1
# 0=启发式（可能拒绝fork）, 1=总是允许, 2=严格限制

# 监控 CoW 内存使用
redis-cli INFO stats | grep latest_fork_usec
```

[src: raw/ingested/2技术/redis/Redis_KeyDB运维问题速查.md]

## Related Pages
- [[性能优化]]
- [[Redis持久化问题]]
- [[Redis性能问题]]
- [[Redis可观测性项目]]
- [[内存管理]]

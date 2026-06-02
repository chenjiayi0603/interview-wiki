# Redis/KeyDB 运维问题速查

> 连接/性能/内存/数据一致性/持久化/集群/安全等运维问题排查与根因分析。按生产案例组织，每个案例包含现象 → 排查 → 根因 → 解决。

---

## 一、连接与网络问题

### 案例1：maxclients 耗尽，新连接被拒

**现象**：
- 客户端报 `max number of clients reached`
- `redis-cli INFO clients` 显示 `connected_clients` 接近 `maxclients`

**排查**：
```bash
redis-cli INFO clients
cat /proc/`pidof redis-server`/limits | grep "open files"
# 系统 fd 限制不足时，即使 maxclients 设很大也无效
```

**根因**：
```
Redis 单线程 + 多路复用，每个连接占一个 fd。
maxclients ≤ ulimit -n - 32（留 32 给内部 fd 用）
默认 maxclients=10000，但系统 ulimit -n 可能只有 1024 → 实际最多 992 连接
```

**解决**：
```bash
# 1. 增大系统 fd 限制
echo "* soft nofile 65535" >> /etc/security/limits.conf
echo "* hard nofile 65535" >> /etc/security/limits.conf

# 2. 增大 Redis maxclients
redis-cli CONFIG SET maxclients 30000
# 或写入 redis.conf: maxclients 30000
```

---

### 案例2：protected-mode 导致远程连接被拒

**现象**：
- 远程客户端 `Connection refused`，本地回环正常
- 日志：`# Deny connection from xx.xx.xx.xx due to protected mode`

**排查**：
```bash
redis-cli CONFIG GET protected-mode  # → yes
redis-cli CONFIG GET bind            # → 127.0.0.1（只有本地）
redis-cli CONFIG GET requirepass     # → (空，无密码)
```

**根因**：
```
Redis 3.2+ 默认 protected-mode=yes：
- 只监听 127.0.0.1 时自动开启保护
- 无密码且无 bind 显式配置时，拒绝所有远程连接
- 目的是防止未授权访问
```

**解决**：
```bash
# 方案A：设置密码（推荐）
requirepass your-strong-password

# 方案B：显式 bind 内网 IP
bind 0.0.0.0  # 或指定内网 IP
# 注：显式 bind 后 protected-mode 自动关闭

# 方案C：关闭保护（不推荐）
protected-mode no
```

---

### 案例3：连接泄漏导致 CLOSE_WAIT 堆积

**现象**：
- 客户端连接数持续增长，不下降
- `netstat -antp | grep CLOSE_WAIT` 数量很大
- 最终达到 maxclients 上限，新请求被拒

**排查**：
```bash
# 查看 CLOSE_WAIT 数量
netstat -antp | grep CLOSE_WAIT | wc -l

# 查看哪些进程的 CLOSE_WAIT
netstat -antp | grep CLOSE_WAIT

# 确认 Redis 端连接数
redis-cli INFO clients | grep connected_clients
```

**根因**：
```
客户端未正确关闭连接：
- 使用 Jedis 等客户端时未调用 close()
- 连接池泄漏：borrow 后未 return
- 异常分支未在 finally 中释放连接
- 服务端已关闭连接，但客户端未感知（没有心跳/keepalive）

CLOSE_WAIT 状态说明：
- 被动关闭方收到 FIN 后进入 CLOSE_WAIT
- 如果应用程序不调用 close()，会一直停留在 CLOSE_WAIT
```

**解决**：
```java
// Jedis 正确用法（try-with-resources / try-finally）
try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value");
} // 自动归还

// 或手动 finally
Jedis jedis = null;
try {
    jedis = pool.getResource();
    jedis.set("key", "value");
} finally {
    if (jedis != null) jedis.close();  // 归还到池
}
```

```bash
# 服务端配置超时，主动关闭空闲连接
redis-cli CONFIG SET timeout 300  # 300 秒空闲超时
```

---

### 案例4：TCP backlog 满导致连接被内核丢弃

**现象**：
- 高并发下部分客户端连接超时
- Redis 日志正常，但客户端 `connect` 超时
- `netstat -s | grep overflowed` 显示握手溢出

**排查**：
```bash
# 查看 TCP 扩展信息
netstat -s | grep -i "overflow\|drop"
# 如果 "TCPBacklogDrop" 或 "listen queue overflow" 持续增长

# 查看当前 backlog
redis-cli CONFIG GET tcp-backlog
```

**根因**：
```
TCP 三次握手：
1. 客户端 SYN → 服务端半连接队列（SYN Queue）
2. 服务端 SYN+ACK → 客户端（收到后进入全连接队列 Accept Queue）
3. backlog 控制 Accept Queue 的大小
4. 队列满时，内核直接丢弃 SYN 或 RST

tcp-backlog 默认 511，但受系统 somaxconn 限制
```

**解决**：
```bash
# 同时调整两个参数
redis-cli CONFIG SET tcp-backlog 1024

# 系统级调整
echo 1024 > /proc/sys/net/core/somaxconn
# 持久化：net.core.somaxconn=1024 → /etc/sysctl.conf
```

---

## 二、性能与延迟问题

### 案例1：慢命令阻塞导致所有请求排队

**现象**：
- 响应延迟突然飙升（从 1ms → 几秒）
- QPS 断崖式下降
- CPU 使用率不高但请求超时

**排查**：
```bash
# 1. 慢查询日志（关键入口）
redis-cli SLOWLOG GET 10
# 查看最近 10 条慢命令
# 注意：SLOWLOG 只记录执行时间 > slowlog-log-slower-than 的命令

# 2. 查看命令耗时统计
redis-cli INFO commandstats
# cmdstat_keys:calls=1,usec=5231000,usec_per_call=5231000.00

# 3. 立即诊断当前正在执行的命令
redis-cli CLIENT LIST
# 查看 cmd 列，或 redis-cli --monitor 实时抓取

# 4. 设置更低的慢查询阈值
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms
```

**根因**：
```
Redis 单线程模型：所有命令串行执行
一个慢命令阻塞 → 后续所有请求排队等待

典型慢命令：
  KEYS *              → O(N)，遍历所有 key（千万级 key 时阻塞秒级）
  SMEMBERS big_set    → O(N)，返回集合所有元素
  SORT big_list       → O(NlogN+M)，排序操作
  FLUSHALL/FLUSHDB    → O(N)，清空所有数据
  ZRANGE big_zset     → O(logN+M)，大范围返回
  DEL big_key         → O(N)，删除大 key（同步阻塞！）
```

**解决**：
```bash
# 禁用危险命令，用替代方案
# redis.conf 中重命名命令
rename-command KEYS ""
rename-command FLUSHALL ""
rename-command FLUSHDB ""

# 替代方案
# KEYS *          → SCAN 0 COUNT 100（游标式迭代）
# SMEMBERS big    → SSCAN big 0 COUNT 100
# GET big_key     → 拆分为多个小 key
# DEL big_key     → UNLINK big_key（异步删除，Redis 4.0+）

# 大 key 删除使用 UNLINK（不阻塞主线程）
redis-cli UNLINK my_big_key
```

---

### 案例2：BGSAVE fork + CoW 导致延迟飙升

**现象**：
- 周期性的延迟抖动，和 RDB 快照/AOF 重写时间吻合
- `latest_fork_usec` 值很大（毫秒级）
- 内存越大延迟越明显

**排查**：
```bash
# 查看 fork 耗时
redis-cli INFO stats | grep latest_fork_usec
# latest_fork_usec:523000  ← 523ms！阻塞主线程

# 查看最近 BGSAVE 状态
redis-cli INFO persistence | grep rdb
# rdb_bgsave_in_progress:0
# rdb_last_bgsave_time_sec:3

# 检查 THP 是否开启
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never  ← THP 开启，会放大 CoW
```

**根因**：
```
fork + CoW 延迟的 3 阶段：

阶段1：fork 系统调用（阻塞主线程）
  - 复制页表（内存越大，耗时越长）
  - 内存 10GB → fork 耗时约 100-300ms
  - fork 执行期间主线程**完全阻塞**
  
阶段2：子进程 BGSAVE（后台）
  - 子进程与父进程共享内存页（只读）
  - 父进程继续服务，修改数据 → 触发 CoW
  - 写量越大，CoW 复制的页面越多
  - 延迟来源：缺页中断 + 页面复制

阶段3：子进程结束
  - 释放内存页
  - 可能触发大块内存回收

THP 放大 CoW 的原理：
  正常：4KB 页 → CoW 时复制 4KB（父进程写 1 字节也只需复制 4KB）
  THP：2MB 页 → CoW 时复制 2MB（放大 512 倍！）
  → 父进程写 1 个字节，内核复制 2MB → 延迟从 μs 级飙升到 ms 级
```

**解决**：
```bash
# 1. 关闭 THP（最关键且有效的手段！）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# 持久化：在 /etc/rc.local 中添加，或用 systemd 服务

# 2. 验证效果
redis-cli INFO stats | grep latest_fork_usec
# 关闭 THP 后 fork 耗时通常降低 50-80%

# 3. 减小 fork 频率
# 增大 RDB save 间隔
save 900 1      # 900 秒内至少 1 次修改
save 300 10     # 300 秒内至少 10 次修改
save 60 10000   # 60 秒内至少 10000 次修改

# 4. 使用子进程方式
# 用 BGSAVE 代替 SAVE（SAVE 是前台阻塞，BGSAVE 是后台 fork）
```

---

### 案例3：Swap 导致延迟 5 个数量级飙升

**现象**：
- 延迟从稳定的微秒级突然跳到毫秒甚至秒级
- QPS 大幅下降但没有慢查询
- 内存碎片率 < 1（反常，因为用了 Swap）

**排查**：
```bash
# 1. 检查 Swap 使用（关键！）
cat /proc/`pidof redis-server`/status | grep Swap
# Swap: 102400 kB  ← 使用了 100MB Swap，危险！

# 2. 查看内存碎片率（< 1 通常意味着用了 Swap）
redis-cli INFO memory | grep mem_fragmentation_ratio
# mem_fragmentation_ratio:0.95  ← < 1，强烈提示 Swap

# 3. 系统级 Swap 使用
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15G         14G        300M         ...         ...         ...
# Swap:          2.0G        1.2G        800M    ← Swap 被大量使用
```

**根因**：
```
延迟对比：
  内存访问：  ~100ns
  SSD 访问：  ~100μs（慢 1000 倍）
  HDD 访问：  ~10ms（慢 100000 倍）

Swap 本质：内核将 Redis 进程的内存页换出到磁盘
当 Redis 访问被换出的内存页 → 缺页中断 → 从磁盘读回

延迟从 100ns → 10ms（5 个数量级的差异！）
Redis 主线程阻塞在 page fault → 所有请求排队

为什么 mem_fragmentation_ratio < 1？
  物理内存不足 → 内核换出内存页 → RSS 减小
  used_memory（实际数据）> used_memory_rss（驻留内存）
  → 比值 < 1
```

**解决**：
```bash
# 1. 立即处理（临时）
# 增大 maxmemory，限制 Redis 内存使用
redis-cli CONFIG SET maxmemory 8gb
# 同时配置淘汰策略
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 2. 根本解决
# - 增加物理内存
# - 减小 maxmemory，为系统预留足够内存
# - 关闭 Swap（或减小 swappiness）
echo 10 > /proc/sys/vm/swappiness  # 默认 60，越低越不倾向 Swap
# 持久化：vm.swappiness=10 → /etc/sysctl.conf

# 3. 监控预警
# 监控 mem_fragmentation_ratio < 1 时告警
# 监控 Swap 使用量 > 0 时告警
```

---

### 案例4：大 Key 操作阻塞主线程

**现象**：
- 单个命令延迟很高（秒级）
- 操作特定 key 时必慢
- 网络出口带宽被打满

**排查**：
```bash
# 1. 扫描大 Key
redis-cli --bigkeys
# Biggest string found so far '"product:1001"' with 52428800 bytes
# Biggest  hash  found so far '"user:followers"' with 1000000 fields

# 2. 查看具体 key 大小
redis-cli MEMORY USAGE product:1001
# (integer) 52500000  ← 约 52MB

# 3. 查看 key 类型
redis-cli TYPE user:followers
# hash

redis-cli HLEN user:followers  
# (integer) 1000000  ← 100 万 field
```

**根因**：
```
大 Key 的危害：

1. 操作耗时：
   - HGETALL 100 万 field → 序列化 + 网络传输耗时秒级
   - DEL 大 key → 同步删除 O(N)，阻塞主线程
   - 大 value（>10MB）序列化/反序列化消耗大量 CPU

2. 网络带宽：
   - 一次读取 50MB → 带宽瞬间打满
   - 千兆网卡理论 125MB/s，一次大 key 占 40% 带宽

3. 主从复制：
   - 大 key 拆分成多条命令 → 复制缓冲区积压
   - 从节点消费跟不上 → 主节点断开从节点连接
```

**解决**：
```bash
# 1. 使用 UNLINK 异步删除
redis-cli UNLINK product:1001  # 后台线程删除，不阻塞主线程

# 2. 拆分大 Key
# Hash: 100 万 field → 拆为 100 个 hash，每个 1 万 field
# 原：user:followers{1000000}
# 拆：user:followers:0 ~ user:followers:99

# 3. 压缩大 value
# 应用层压缩（gzip/snappy）后写入，读取后解压
```

---

### 案例5：AOF fsync 阻塞主线程

**现象**：
- 写延迟周期性飙升
- 开启 `appendfsync always` 时尤其明显
- 磁盘 IO 利用率高

**排查**：
```bash
redis-cli CONFIG GET appendfsync
# appendfsync: always  ← 每次写都 fsync，磁盘慢时阻塞

# 查看持久化状态
redis-cli INFO persistence | grep aof
# aof_delayed_fsync:500  ← fsync 阻塞次数持续增长
```

**根因**：
```
AOF fsync 策略：

always：
  每个写命令执行后都调用 fsync()
  主线程等待 fsync 完成 → 磁盘 IO 慢时阻塞
  性能下降 50%+，SSD 上也有明显影响

everysec（推荐）：
  后台线程每秒 fsync 一次
  主线程不等待 fsync 完成
  最多丢 1 秒数据

no：
  由操作系统决定何时刷盘
  宕机可能丢 30 秒以上数据
```

**解决**：
```bash
# 推荐配置：everysec
redis-cli CONFIG SET appendfsync everysec

# 如果磁盘 IO 极差（如 HDD），可考虑 no
# redis-cli CONFIG SET appendfsync no

# 避免 AOF 重写期间 fsync 阻塞
redis-cli CONFIG SET no-appendfsync-on-rewrite yes
# 重写期间不 fsync，避免 fork + fsync 双重阻塞
```

---

## 三、内存问题

### 案例1：OOM 进程被系统 kill

**现象**：
- Redis 进程突然消失
- `dmesg` 显示 `Out of memory: Kill process`
- 日志：`redis-server invoked oom-killer`

**排查**：
```bash
# 查看系统日志中的 OOM 信息
dmesg | grep -i "out of memory"
dmesg | grep -i "redis"

# 查看 Redis 退出前内存使用
# 如果未配置 maxmemory，Redis 会无限增长直到 OOM

# 检查 overcommit 配置
cat /proc/sys/vm/overcommit_memory
# 0: 默认，fork 时检查内存是否足够
# 1: 总是允许 overcommit（推荐 Redis 使用）
```

**根因**：
```
OOM 触发路径：

场景A：未设置 maxmemory
  Redis 内存无限增长 → 物理内存耗尽 → OOM killer 杀掉 Redis
  
场景B：fork 时触发
  overcommit_memory=0 → fork 时检查内存
  即使内存未满，但加上子进程需要的 CoW 内存
  → 被认为"内存不足" → fork 失败或 OOM

场景C：内存碎片 + 复制缓冲区等额外开销
  used_memory 未超 maxmemory，但 RSS 已超物理内存
```

**解决**：
```bash
# 1. 必须设置 maxmemory（预留 20-30% 给系统和其他进程）
redis-cli CONFIG SET maxmemory 8gb  # 物理内存 10GB 时

# 2. 开启 overcommit（避免 fork 时 OOM）
echo 1 > /proc/sys/vm/overcommit_memory
# 持久化：vm.overcommit_memory=1 → /etc/sysctl.conf

# 3. 配置淘汰策略（避免达到 maxmemory 后写入失败）
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

---

### 案例2：内存碎片率高

**现象**：
- `used_memory_rss` 远大于 `used_memory`
- `mem_fragmentation_ratio > 1.5`
- 内存占用不合理地高

**排查**：
```bash
redis-cli INFO memory | grep -E "used_memory|mem_fragmentation"
# used_memory: 4294967296          ← 数据实际占用 4GB
# used_memory_rss: 7516192768      ← RSS 7GB
# mem_fragmentation_ratio: 1.75    ← 碎片率 1.75，偏高

# 查看大 Key 删除历史
redis-cli INFO stats | grep evicted_keys
```

**根因**：
```
jemalloc 分配器机制：

jemalloc 将内存划分为固定 size class：
8, 16, 32, 48, 64, 80, 96, 112, 128, ...
                    ↓
需要 30 字节 → 分配 32 字节（2 字节内部碎片）
需要 33 字节 → 分配 48 字节（15 字节内部碎片）

碎片产生原因：
1. 内部碎片：size class 对齐导致
2. 外部碎片：分配和释放模式不匹配
3. 大 Key 删除后：内存页未归还 OS（jemalloc 缓存）

碎片率判断：
  ≈ 1：理想状态
  1.0~1.5：正常
  > 1.5：碎片较多，需要处理
  < 1：已使用 Swap（危险！见案例3）
```

**解决**：
```bash
# 1. 启用自动碎片整理（Redis 4.0+）
redis-cli CONFIG SET activedefrag yes
redis-cli CONFIG SET active-defrag-threshold-lower 10   # 碎片率 > 1.1 时开始
redis-cli CONFIG SET active-defrag-threshold-upper 100  # 碎片率 > 2.0 时全力
redis-cli CONFIG SET active-defrag-cycle-min 5          # CPU 占用最小 5%
redis-cli CONFIG SET active-defrag-cycle-max 75         # CPU 占用最大 75%

# 2. 重启（碎片整理完的最直接方式）
# 主从架构：重启从节点，切换主从，再重启原主

# 3. 手动触发内存整理
redis-cli MEMORY PURGE  # 尝试释放 jemalloc 缓存
```

---

### 案例3：淘汰策略不当导致写入失败

**现象**：
- 达到 maxmemory 后写入返回 OOM 错误
- `OOM command not allowed when used memory > maxmemory`

**排查**：
```bash
redis-cli CONFIG GET maxmemory
# maxmemory: 8589934592  ← 8GB

redis-cli CONFIG GET maxmemory-policy
# maxmemory-policy: noeviction  ← 不淘汰策略！写入会失败

redis-cli INFO memory | grep used_memory
# used_memory: 8589934592  ← 已用完 maxmemory
```

**根因**：
```
maxmemory-policy=noeviction（默认）：
  内存满时写入操作返回错误，不淘汰任何 key
  适合"不允许丢数据"的场景，但缓存场景应使用淘汰策略

淘汰策略选型：
| 策略 | 含义 | 适用场景 |
|------|------|---------|
| noeviction（默认） | 不淘汰，写入返回 OOM | 不允许丢失数据 |
| allkeys-lru | 所有 Key 中淘汰最近最少使用 | 通用缓存（推荐） |
| volatile-lru | 带 TTL 的 Key 中淘汰最近最少使用 | 有 TTL 的缓存 |
| allkeys-lfu | 所有 Key 中淘汰最不经常使用 | 访问频率差异大 |
| volatile-lfu | 带 TTL 的淘汰最不经常使用 | 有 TTL + 频率差异大 |
| allkeys-random | 随机淘汰 | 访问均匀 |
| volatile-ttl | 淘汰 TTL 最小的 | 优先淘汰快过期的 |

Redis 淘汰采样实现：
  1. 默认每次采样 5 个 Key（maxmemory-samples）
  2. 从采样结果中选"最优"的 Key 淘汰
  3. 如未达标，继续采样淘汰
  4. 采样数越大越精确，CPU 消耗也越大
```

**解决**：
```bash
# 缓存场景：推荐 allkeys-lru
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG SET maxmemory-samples 10  # 提高精度

# 有时效数据场景：volatile-lru
redis-cli CONFIG SET maxmemory-policy volatile-lru

# 访问频率差异大：allkeys-lfu
redis-cli CONFIG SET maxmemory-policy allkeys-lfu
```

---

### 案例4：fork 导致 OOM

**现象**：
- 执行 BGSAVE 或 BGREWRITEAOF 时 Redis 被 kill
- 日志：`Can't save in background: fork: Cannot allocate memory`

**排查**：
```bash
# 检查 overcommit 设置
cat /proc/sys/vm/overcommit_memory
# 0 → fork 时内核检查内存是否充足，CoW 可能被认为不够

# 查看内存使用
redis-cli INFO memory | grep used_memory_rss
```

**根因**：
```
fork 的 COW 内存计算：
  overcommit_memory=0 时，fork 需要"保证"父子进程都能运行
  内核估算：父进程 RSS + 子进程潜在最大 CoW 量 ≤ 物理内存
  如果接近内存上限 → fork 失败 → BGSAVE 无法执行

overcommit_memory 三种模式：
  0：启发式 overcommit（默认） → fork 时可能失败
  1：总是允许 overcommit → fork 总是成功（推荐 Redis）
  2：限制 overcommit → 超过 swap+RAM*比例 则拒绝
```

**解决**：
```bash
# 立即生效
echo 1 > /proc/sys/vm/overcommit_memory

# 持久化
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
sysctl -p

# 同时减小 maxmemory，留出 CoW 空间
redis-cli CONFIG SET maxmemory 70%  # 最多用 70% 物理内存
```

---

## 四、数据一致性问题

### 案例1：主从异步复制延迟

**现象**：
- 写主库后立即读从库，读不到最新数据
- 从库数据落后于主库

**排查**：
```bash
# 查看主从复制状态
redis-cli INFO replication
# role:master
# connected_slaves:1
# slave0:ip=xx.xx.xx.xx,port=6379,state=online,lag=5  ← lag=5 秒延迟

# 从库查看
redis-cli INFO replication
# role:slave
# master_last_io_seconds_ago:5  ← 5 秒没有和主库通信
# master_repl_offset: 100000
# slave_repl_offset: 95000       ← 落后 5000 条命令
```

**根因**：
```
Redis 复制是异步的：
  主节点写入后立即返回客户端
  从节点异步拉取复制缓冲区中的数据
  不等待从节点确认（非半同步）

延迟原因：
  1. 网络延迟（主从跨机房、跨 AZ）
  2. 从节点处理慢（单线程重放）
  3. 大事务/大 key 复制
  4. 主节点网络出口带宽打满

主从复制是异步的。即使设置 min-replicas-to-write：
  - 主节点检查从节点数量和延迟是否满足阈值
  - 满足则允许写入并立即返回客户端
  - 不会等待从节点确认写入完成再响应
```

**解决**：
```bash
# 1. 强制最小从节点数量（写后至少 N 个从节点确认在线）
min-replicas-to-write 1
min-replicas-max-lag 10
# 如果从节点 < 1 或延迟 > 10秒，主节点拒绝写入

# 2. 业务层处理
# 对一致性要求高的读，强制读主库

# 3. 网络优化
# 主从尽量同机房部署
# 跨机房用专线/高带宽连接
```

---

### 案例2：过期键在主从不一致（3.2 之前）

**现象**：
- 从库读到已过期的 key
- 主库正常返回 nil，从库返回旧数据

**排查**：
```bash
redis-cli INFO server | grep redis_version
# redis_version: 3.0.6  ← 3.2 之前版本

# 在主库设置带 TTL 的 key，等待过期后从库仍能读到
```

**根因**：
```
Redis 过期键删除策略：
  1. 惰性删除：访问 key 时检查是否过期，过期则删除
  2. 定时采样删除：每 100ms 随机采样 20 个带 TTL 的 key，删除过期的

3.2 之前的问题：
  - 从节点不会主动删除过期键
  - 只等主节点发来 DEL 命令
  - 如果主节点还未触发惰性/定时删除
  - → 从节点读到过期但仍存在的脏数据

3.2+ 修复：
  - 从节点对过期 key 返回空（逻辑上不可见）
  - 但物理上仍然存在，等待主节点 DEL 或逐出
  - 解决了读不一致的问题
```

**解决**：
```bash
# 升级 Redis 到 3.2+（推荐）
# 3.2+ 从节点逻辑过期，返回 nil

# 临时方案：主库定期扫描并清理过期 key
# 使用 SCAN + TTL 主动删除
```

---

### 案例3：复制缓冲区溢出导致全量同步

**现象**：
- 从节点断连后重连，触发全量同步（而非增量）
- 全量同步期间主节点 CPU/内存/带宽飙升
- 主从延迟大幅增加

**排查**：
```bash
# 查看复制 backlog 状态
redis-cli INFO replication
# repl_backlog_active:1
# repl_backlog_size:1048576       ← backlog 只有 1MB（默认）
# repl_backlog_first_byte_offset:100
# repl_backlog_histlen:1048576

# 从节点重连时触发全量同步
# sync_full 计数增加
redis-cli INFO stats | grep sync_full
```

**根因**：
```
repl_backlog 机制：
  主节点维护一个环形缓冲区（默认 1MB）
  记录所有写命令，供从节点增量同步

溢出场景：
  从节点断连时间过长
  断连期间主节点持续写入
  repl_backlog 被新数据覆盖
  → 从节点重连时找不到需要的 offset
  → 只能全量同步（重传 RDB）
  → 全量同步期间从节点不可用

client-output-buffer-limit（另一个缓冲区）：
  主节点为每个从节点连接的 socket 输出缓冲区
  从节点消费太慢 → 命令积压 → 超阈值 → 主节点断开该从节点
  注意：每个从节点独立，互不影响
  "主节点是主动推送，但需要从节点接得住"
```

**解决**：
```bash
# 1. 增大 repl_backlog_size
# 建议值：平均写入速率(MB/s) × 预期最大断连秒数 × 2
repl-backlog-size 256mb

# 2. 增大 client-output-buffer-limit
client-output-buffer-limit replica 512mb 256mb 60
# 硬限制 512MB，软限制 256MB/60秒

# 3. 避免大 key 复制（hash field 建议 < 10000）
# 大 hash 同步时会拆成大量 hset 命令，塞满复制缓冲区
```

---

### 案例4：大 hash 复制导致缓冲区溢出

**现象**：
- 主从复制频繁断开重连
- 日志：`client-output-buffer-limit` 触发断连
- 监控显示复制缓冲区持续高水位

**排查**：
```bash
# 查看从节点连接状态
redis-cli CLIENT LIST | grep slave
# 看 buf 和 obl 列是否接近限制

# 检查大 key
redis-cli --bigkeys
# hash 类型的 key field 数量巨大
```

**根因**：
```
大 hash 复制过程：
  主节点无法用一个命令完整复制大 hash
  → 拆分成大量 hset（或 hmset）命令逐条同步
  → 这些命令积压在 client-output-buffer-limit 中
  → 从节点处理跟不上
  → 缓冲区满 → 主节点断开该从节点

注意两个缓冲区的区别：
  repl_backlog：主节点缓存写命令历史，用于增量复制。满了→全量同步
  client-output-buffer-limit replica：主节点为每个从节点连接的输出缓冲区。满了→断开从节点
  → 不是所有从节点共用一份，而是每个从节点各自独立
```

**解决**：
```bash
# 1. 避免超大 hash（单个 key field 建议 < 10000）
# 拆分：user:followers:0 ~ user:followers:99

# 2. 调大缓冲区
client-output-buffer-limit replica 512mb 256mb 60

# 3. 热 key 分片、热点分散
# 业务层将热点数据分散到多个 key

# 4. 分批迁移大 key
# 生产上热 key 分片效果更好
```

---

### 案例5：缓存与 DB 双写不一致

**现象**：
- Redis 中数据与 MySQL 中数据不一致
- 旧数据长期存在于缓存中

**排查**：
```bash
# 对比缓存和数据库值
redis-cli GET user:1001
# "name_old"

# MySQL
SELECT name FROM users WHERE id=1001;
# "name_new"  ← 不一致！
```

**根因**：
```
Cache-Aside 模式的并发问题：

时刻T1：线程A 读缓存 miss → 读DB得到旧值 value_old
时刻T2：线程B 更新DB为 value_new → 删除缓存（成功）
时刻T3：线程A 将 value_old 写入缓存  ← 旧值覆盖！

关键点：线程A 读DB 和 写缓存 之间不是原子的
线程B 在这之间更新DB+删缓存，但仍被旧值覆盖
```

**解决**：

##### 方案1：延迟双删（简单但无法 100% 保证）

```
1. 删缓存
2. 更新DB
3. sleep(N ms)  ← 等待并发"读旧DB+写缓存"的请求完成
4. 再次删缓存  ← 清除可能被写回的旧值
```

延迟 N 应略大于一次"读DB+写缓存"的耗时。缺点：引入了应用层延迟，极端并发下仍有覆盖可能。

---

##### 方案2：基于 binlog 的异步更新（推荐，最终一致性）

通过 Canal/Debezium 监听 MySQL binlog，异步删除/更新缓存，实现最终一致性。

**架构总览（为什么是这个链路）**：

```
┌─────────────────────────────────────────────────────────────────────┐
│ 核心链路：MySQL → Canal(伪装slave拉取binlog) → 消费者(删缓存) → Redis │
└─────────────────────────────────────────────────────────────────────┘

步骤分解：
                                   ③ Canal 把binlog事件
                                   发给消费者（TCP/Kafka）
                                      │
   ① 业务写入DB         ② Canal 伪装成         ④ 消费者解析事件       ⑤ 删除/更新
                         MySQL slave            提取表名+主键          对应缓存
                        拉取 binlog
  ┌──────────┐  写入  ┌──────────────┐  转发  ┌──────────────┐  删除  ┌──────────┐
  │  应用服务  │ ────→ │  Canal       │ ─────→ │  消费者服务   │ ─────→ │  Redis   │
  │ update DB │       │  (伪装slave)  │        │ parse event  │        │  cache   │
  └──────────┘       │  拉binlog    │        │ filter table │        └──────────┘
                     └──────────────┘        └──────────────┘
                           │                        │
                     Canal 本质就是             消费者可以内嵌
                     一个"binlog转发器"         Canal Client 直连
                                              也可以走 Kafka 解耦
```

**关键理解**：
```
Canal 的角色：伪装成 MySQL 的从库（slave），不断拉取 binlog，然后把 binlog 事件转发出去。
它本身不做缓存删除——它只是一个"管道"。

消费者的角色：从 Canal（或 Kafka）拿到 binlog 事件，解析出"哪张表的哪条记录被改了"，
然后决定删哪个 Redis key。缓存删除的逻辑在这里。

  ⚠ 消费者 = 你自己开发的常驻进程，不是现成工具
     因为只有你的业务知道"user 表 id=1001 改了 → 删 Redis key user:1001"
     没有任何通用组件能替你决定这条规则

队列（Kafka）的角色：可选的中转站。Canal 把事件丢进 Kafka 就走，
消费者从 Kafka 拉，两边互不影响。小规模场景可以不用队列，消费者直接
通过 TCP 连 Canal。

三种部署模式：
  模式A（无队列）：Canal ──TCP──→ 你的消费者进程 ──→ Redis
  模式B（有队列）：Canal ──→ Kafka ──→ 你的消费者进程 ──→ Redis
  模式C（Debezium）：Debezium ──→ Kafka ──→ 你的消费者进程 ──→ Redis

所以流程是：
  应用更新MySQL → Canal拉取binlog → 消费者解析 → 删除Redis缓存
                        ↑                            ↑
                  转发 binlog 事件               真正做业务判断的地方
```

**选型对比：Canal vs Debezium**

| 维度 | Canal（阿里） | Debezium（Red Hat） |
|------|-------------|-------------------|
| 部署方式 | 独立 Java 服务 | Kafka Connect Connector |
| 数据输出 | TCP 直连 / MQ (Kafka/RocketMQ) | Kafka（必须依赖 Kafka） |
| 增量快照 | 支持 | 支持 |
| 监控 | 自带 admin 界面 | Kafka Connect REST API + JMX |
| 社区 | 中文为主 | 国际社区活跃 |
| 适用场景 | 中小团队、轻量部署 | 已有 Kafka 的大数据场景 |

---

**方案2a：Canal 部署（轻量，推荐中小团队）**

**1. 开启 MySQL binlog**

```ini
# my.cnf
[mysqld]
log-bin=mysql-bin          # 开启 binlog
binlog-format=ROW          # ROW 格式才能看到修改前后的值
server-id=1                # 集群内唯一
expire_logs_days=7         # 保留 7 天
binlog-do-db=your_db       # 只监听指定库（可选）
```

**2. 创建 Canal 专用 MySQL 账号**

```sql
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal_pass';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

**3. 部署 Canal Server（docker-compose）**

```yaml
# docker-compose.yml
version: '3'
services:
  canal-server:
    image: canal/canal-server:v1.1.6
    ports:
      - "11111:11111"   # Canal TCP 端口
    environment:
      - canal.instance.mysql.slaveId=100
      - canal.instance.master.address=mysql-host:3306
      - canal.instance.dbUsername=canal
      - canal.instance.dbPassword=canal_pass
      - canal.instance.filter.regex=your_db\\..*   # 监听表
      - canal.mq.topic=redis-cache-topic
      - canal.server.mode=tcp   # 直连模式，不依赖 MQ
    volumes:
      - ./canal/conf:/admin/canal-server/conf
      - ./canal/logs:/admin/canal-server/logs
    restart: always
```

**4. 编写消费者（Java 示例）**

```java
// 依赖：com.alibaba.otter:canal.client
public class RedisCacheSyncConsumer {
    public static void main(String[] args) {
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("canal-server", 11111),
            "redis-cache-topic",  // destination
            "", ""                // 用户名/密码
        );
        
        connector.connect();
        connector.subscribe("your_db\\.(user|product|order)");
        connector.rollback();
        
        while (true) {
            Message message = connector.getWithoutAck(100, 2, TimeUnit.SECONDS);
            long batchId = message.getId();
            if (batchId == -1 || message.getEntries().isEmpty()) {
                Thread.sleep(100);
                continue;
            }
            
            for (Entry entry : message.getEntries()) {
                if (entry.getEntryType() != EntryType.ROWDATA) continue;
                
                RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
                String tableName = entry.getHeader().getTableName();
                
                for (RowData rowData : rowChange.getRowDatasList()) {
                    String key = buildCacheKey(tableName, rowData);
                    
                    switch (rowChange.getEventType()) {
                        case UPDATE:
                        case DELETE:
                            jedis.del(key);  // 删除缓存，让下次请求重新加载
                            log.info("Cache invalidated: {}", key);
                            break;
                        case INSERT:
                            // INSERT 通常不需要删缓存
                            break;
                    }
                }
            }
            connector.ack(batchId);  // 确认消费
        }
    }
    
    private static String buildCacheKey(String table, RowData row) {
        // 根据表名和主键构建 Redis key，例如 user+id=1001 → "user:1001"
        for (Column col : row.getAfterColumnsList()) {
            if (col.getIsKey()) {  // 主键列
                return table + ":" + col.getValue();
            }
        }
        return null;
    }
}
```

**5. Canal 关键配置**

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `canal.instance.filter.regex` | 监听表 | `your_db\\.(user\|product)` |
| `canal.instance.filter.black.regex` | 排除表 | `your_db\\.(tmp\|backup)` |
| `canal.server.mode` | 输出模式 | `tcp`（直连）或 `kafka`（MQ） |
| `canal.instance.batch.size` | 批量拉取条数 | 2048 |

---

**方案2b：Debezium 部署（已有 Kafka 的场景）**

**1. 部署 Kafka + Kafka Connect + Debezium**

```yaml
# docker-compose.yml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  connect:
    image: debezium/connect:2.5
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect-configs
      OFFSET_STORAGE_TOPIC: connect-offsets
      STATUS_STORAGE_TOPIC: connect-statuses
    depends_on:
      - kafka
```

**2. 注册 MySQL Connector**

```bash
curl -i -X POST -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  http://localhost:8083/connectors/ \
  -d '{
    "name": "mysql-connector",
    "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector",
      "database.hostname": "mysql-host",
      "database.port": 3306,
      "database.user": "debezium",
      "database.password": "debezium_pass",
      "database.server.id": 200,
      "database.server.name": "my-app",
      "database.include.list": "your_db",
      "table.include.list": "your_db.user,your_db.product",
      "database.history.kafka.bootstrap.servers": "kafka:9092",
      "database.history.kafka.topic": "schema-changes.my-app",
      "value.converter.schemas.enable": false
    }
  }'
```

**3. 消费者（Go 示例）**

```go
package main

import (
    "context"
    "encoding/json"
    "github.com/go-redis/redis/v8"
    "github.com/segmentio/kafka-go"
    "log"
)

type ChangeEvent struct {
    Op     string                 `json:"op"`     // c=create, u=update, d=delete
    After  map[string]interface{} `json:"after"`
    Source struct {
        Table string `json:"table"`
    } `json:"source"`
}

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "redis:6379"})
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"kafka:9092"},
        Topic:   "my-app.your_db.user",
        GroupID: "redis-cache-sync",
    })

    for {
        msg, _ := reader.ReadMessage(context.Background())
        var event ChangeEvent
        json.Unmarshal(msg.Value, &event)

        if event.Op == "u" || event.Op == "d" {  // update / delete
            id := event.After["id"]
            key := "user:" + fmt.Sprint(id)
            rdb.Del(context.Background(), key)
            log.Printf("Cache invalidated: %s (op=%s)", key, event.Op)
        }
    }
}
```

---

**运维注意事项**

| 注意事项 | 说明 |
|---------|------|
| **binlog 保留天数** | 确保消费者故障恢复时 binlog 未被清理，建议保留 3-7 天 |
| **消费延迟监控** | Canal/Debezium 的消费延迟需告警，延迟过大说明消费者处理不过来 |
| **幂等性设计** | 删除缓存是幂等的，多次删除不影响业务 |
| **批量效果** | 支持批量推送，建议批量删除减少 Redis 网络往返 |
| **全量初始化** | 初次上线时全量扫描 DB 预热缓存，或等业务访问触发加载 |
| **故障恢复** | Canal/Debezium 支持 offset 持久化，重启后从断点继续消费 |
| **MySQL 8.0+** | Canal 需配置 `canal.instance.dbDriver=com.mysql.cj.jdbc.Driver` |
| **云数据库** | 华为云 RDS / 阿里云 RDS 都支持开启 binlog，控制台操作即可 |

```sql
-- 检查 binlog 是否开启
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';

-- 查看当前 binlog 文件位置（Canal/Debezium 会用到）
SHOW MASTER STATUS;
```

---

##### 方案3：读请求强制读主库（短时间内）

```
更新DB后，标记该 key 在 N 秒内强制读 DB
避免读缓存中的旧值
```

实现方式：
1. 更新 DB 时，在 Redis 中设一个短 TTL 标记，如 `read_fallback:user:1001`（TTL=1s）
2. 读请求先检查标记：存在则跳过缓存直接读 DB
3. 标记过期后恢复正常缓存读取
4. 缺点：读 DB 期间延迟增加，多一次 Redis 查询

---

## 五、持久化问题

### 案例1：RDB 快照丢失数据

**现象**：
- Redis 宕机重启后，发现丢失了最近几分钟的数据
- 检查 RDB 文件，最后 save 时间在宕机前

**排查**：
```bash
# 查看 RDB 最后保存时间
redis-cli INFO persistence | grep rdb_last_save_time
# rdb_last_save_time:1617000000
# 与当前时间对比，确认丢失窗口

# 查看 save 配置
redis-cli CONFIG GET save
# save 900 1 300 10 60 10000
```

**根因**：
```
RDB（快照）是"定时拍照"：
  时间线：[save点1] -------- 写入100条 -------- [宕机]
                                                  ↑ 丢失这100条
  save 间隔越大，丢失窗口越大

RDB 本质：某一时刻的全量内存数据二进制序列化
  两次快照之间发生的写入 → 全都丢失
  即使 60 秒 save 一次，也最多丢 60 秒数据

对比 AOF：
  AOF 是追加日志，每个写命令记录到文件
  everysec 策略最多丢 1 秒数据
```

**解决**：
```bash
# 1. 开启 AOF（减少丢失窗口）
appendonly yes
appendfsync everysec  # 最多丢 1 秒数据

# 2. 使用混合持久化（Redis 4.0+，5.0+ 默认开启）
aof-use-rdb-preamble yes
# AOF 重写时生成 RDB 格式数据 + AOF 增量日志
# 恢复时先加载 RDB（快），再回放 AOF（完整）

# 3. 不要仅依赖 RDB
# 重要数据同时开启 AOF
```

---

### 案例2：AOF 文件损坏无法启动

**现象**：
- Redis 重启失败，日志提示 AOF 文件校验错误
- `Bad file format reading the append only file`

**排查**：
```bash
# 检查 AOF 文件
ls -la /var/lib/redis/appendonly.aof

# 检查 Redis 日志
# Catastrophic error: ... Bad file format
```

**根因**：
```
AOF 文件损坏原因：
  1. 异常断电：写入过程中断电，文件尾部不完整
  2. 磁盘坏道：物理存储介质故障
  3. 误删改：人为操作删除了 AOF 文件部分内容
  4. 磁盘空间满：AOF 写入时磁盘满，文件截断

AOF 文件格式：
  每个写命令以 * 开头，以换行分隔
  文件尾部可能不完整（断电时只写了前半条命令）
```

**解决**：
```bash
# 使用 redis-check-aof 修复
redis-check-aof --fix appendonly.aof
# 会截断到最后一个完整命令，丢弃损坏的尾部

# 验证修复结果
redis-check-aof appendonly.aof
# AOF analyzed: size=..., ok_up_to=...
# 确认文件可读

# 备份原文件后再修复
cp appendonly.aof appendonly.aof.bak
redis-check-aof --fix appendonly.aof
```

---

### 案例3：fork 失败导致持久化无法执行

**现象**：
- BGSAVE 或 BGREWRITEAOF 持续失败
- 日志：`Can't save in background: fork: Cannot allocate memory`
- 持久化长时间未执行

**排查**：
```bash
redis-cli INFO persistence
# rdb_bgsave_in_progress:0
# rdb_last_bgsave_status:err  ← 上次 BGSAVE 失败

# 检查系统日志
dmesg | tail -20
# 可能看到 fork 相关错误
```

**根因**：
```
fork 失败原因：
  1. overcommit_memory=0 → 内核估算内存不足拒绝 fork
  2. 内存碎片严重 → 需要大量连续内存
  3. 系统线程数限制 → max_threads / pid_max 耗尽

fork 需要的内存：
  - 复制页表（内存 10GB → 页表约 20MB）
  - CoW 预留（内核需要估算写时复制需要的空间）
```

**解决**：
```bash
# 1. 开启 overcommit
echo 1 > /proc/sys/vm/overcommit_memory

# 2. 减小 maxmemory，留出 CoW 空间
redis-cli CONFIG SET maxmemory 70%  # 物理内存的 70%

# 3. 增大系统 pid_max（如果线程数不够）
echo 65536 > /proc/sys/kernel/pid_max
```

---

### 案例4：磁盘空间不足导致持久化失败

**现象**：
- RDB/AOF 写入失败
- `ENOSPC` 错误
- Redis 日志：`Write error saving DB`

**排查**：
```bash
# 检查磁盘空间
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   50G     0 100%  ← 满了！

# 查看 Redis dir 配置
redis-cli CONFIG GET dir
# 检查该目录所在的磁盘空间

# 检查大文件
du -sh /var/lib/redis/*
# 4.0G    dump.rdb
# 12G     appendonly.aof  ← AOF 文件可能很大
```

**根因**：
```
磁盘空间满的原因：
  1. AOF 文件持续增长，未及时重写
  2. RDB 快照文件过大
  3. 系统日志/其他进程占满磁盘

AOF 重写未及时触发：
  auto-aof-rewrite-percentage 100
  auto-aof-rewrite-min-size 64mb
  AOF 文件增长到上次重写时的 100% 才触发重写
  写入量大时 AOF 可能先撑爆磁盘
```

**解决**：
```bash
# 1. 清理磁盘空间
# 删除不必要的备份文件
# 手动触发 AOF 重写压缩
redis-cli BGREWRITEAOF

# 2. 调整 AOF 重写触发阈值
auto-aof-rewrite-percentage 50   # 增长 50% 就重写
auto-aof-rewrite-min-size 1gb    # 最小 1GB 才重写

# 3. 将持久化文件移到更大的磁盘
# 修改 dir 配置
dir /data/redis  # 指向大容量磁盘

# 4. 使用混合持久化（AOF 重写时生成 RDB 格式，体积更小）
aof-use-rdb-preamble yes
```

---

## 六、集群问题

### 案例1：脑裂导致数据丢失

**现象**：
- 网络抖动后恢复，发现部分数据丢失
- 原主节点降级为从，其上的写入数据没了
- 业务方反馈"写成功的数据查不到了"

**排查**：
```bash
# 查看集群日志中的 failover 记录
# 检查是否有异常的主备切换

redis-cli CLUSTER NODES
# 节点角色变化历史
```

**根因**：
```
脑裂（Split Brain）过程：

网络分区前：
  [主A] ←复制→ [从A']    [主B] ←复制→ [从B']
  
网络分区后：
  分区1：[主A] + [部分客户端]  ← 主A 仍在接受写入
  分区2：[从A'被选为新主A'] + [主B] + [从B'] + [另一部分客户端]
  → 两个分区各自接受写入 → 数据分叉
  → 网络恢复后，旧主A变为从节点，其分区内的写入数据被丢弃！

常见诱因：
  - 主从网络短时丢包/延迟，触发故障转移
  - cluster-node-timeout 设置过短
  - 云环境网络抖动
```

**解决**：
```bash
# 1. min-replicas-to-write（关键防御）
min-replicas-to-write 1
min-replicas-max-lag 10
# 主节点检测不到足够从节点 → 拒绝写入
# 牺牲可用性换一致性

# 2. 增大 cluster-node-timeout
cluster-node-timeout 15000  # 默认 15 秒
# 减少误判故障的概率

# 3. 部署建议
# 三节点以上分布在不同机房/机架
# 避免单机多实例
```

---

### 案例2：槽位未全覆盖导致集群不可用

**现象**：
- 集群写入全部失败
- `(error) CLUSTERDOWN The cluster is down`
- 有节点宕机后整个集群不可用

**排查**：
```bash
redis-cli CLUSTER INFO
# cluster_state:fail  ← 集群不可用
# cluster_slots_assigned:16384
# cluster_slots_ok:15000
# cluster_slots_fail:1384  ← 有槽位不可用

redis-cli CLUSTER NODES
# 查看哪个节点 fail
```

**根因**：
```
Redis Cluster 将 key 空间划分为 16384 个哈希槽：
  slot = CRC16(key) % 16384

cluster-require-full-coverage yes（默认）：
  只要有一个 slot 没有主节点负责，整个集群拒绝写入
  目的是保证数据完整性
  副作用：单点故障导致全集群不可用

槽位未被覆盖的两种情况：
  1. 节点宕机，其负责的 slot 无从节点接管
  2. 初始部署时 slot 未分配完（运维失误）
```

**解决**：
```bash
# 1. 故障时允许部分 slot 不可用（降级）
cluster-require-full-coverage no
# 但注意：访问故障 slot 的 key 会报 MOVED 重定向失败

# 2. 确保每个主节点都有从节点
# 主节点宕机时从节点自动提升

# 3. 修复故障节点后重新分配 slot
redis-cli CLUSTER ADDSLOTS ...
# 或 CLUSTER RESHARD 重新均衡
```

---

### 案例3：槽位迁移失败

**现象**：
- 执行 CLUSTER RESHARD 时卡住或超时
- 迁移过程中断，槽位处于中间状态（MIGRATING/IMPORTING）
- 部分 key 无法正常访问

**排查**：
```bash
redis-cli CLUSTER NODES
# 查看 slot 状态，是否有 MIGRATING/IMPORTING 标志

redis-cli CLUSTER SLOTS
# 检查槽位分配是否正常
```

**根因**：
```
CLUSTER RESHARD 底层是逐 key 迁移：

1. 目标节点: CLUSTER SETSLOT <slot> IMPORTING <source-id>
2. 源节点:   CLUSTER SETSLOT <slot> MIGRATING <target-id>
3. 循环迁移:
   源节点: MIGRATE <target-ip> <target-port> "" 0 5000 KEYS <key1> <key2>...
4. 所有节点: CLUSTER SETSLOT <slot> NODE <target-id>

失败场景：
  - 大 key 迁移超时（MIGRATE 默认超时 5 秒）
  - 网络抖动导致连接中断
  - 源节点和目标节点版本不一致
```

**解决**：
```bash
# 1. 重置中间状态
redis-cli CLUSTER SETSLOT <slot> STABLE

# 2. 拆分大 key 后再迁移
# 大 key 先拆分为多个小 key

# 3. 增大 MIGRATE 超时
# MIGRATE 命令支持指定超时时间 (单位 ms)：
# MIGRATE host port "" 0 10000 KEYS key1 key2  ← 超时 10 秒

# 4. 批量迁移时使用 redis-shake 等工具
# redis-shake 支持断点续传
```

---

### 案例4：故障检测与自动 Failover 延迟

**现象**：
- 主节点宕机后，服务中断了十几秒才恢复
- 从节点晋升为主节点耗时过长

**排查**：
```bash
# 查看 cluster-node-timeout
redis-cli CONFIG GET cluster-node-timeout
# cluster-node-timeout:15000  ← 15 秒

# 查看 Gossip 检测的故障节点
redis-cli CLUSTER NODES
# 看 flags 列是否有 ?(pfail) 或 fail
```

**根因**：
```
Redis Cluster 故障检测流程（Gossip 协议）：

1. 每个节点定期向随机几个节点发送 PING（每秒 10 次）
2. 超过 cluster-node-timeout 未收到 PONG → 标记 PFAIL
3. 超过半数主节点标记某节点为 PFAIL → 升级为 FAIL
4. 该故障主节点的从节点发起选举
5. 从节点获得超过半数主节点投票 → 晋升为新主
6. 新主接管原主的所有 slot

Failover 耗时拆解：
  检测时间 ≈ cluster-node-timeout（默认 15s）
  选举时间 ≈ < 1秒（Gossip 传播 + 投票）
  总计 ≈ 15-16 秒
  
  这是大多数场景下线性的检测等待时间，但在某些情况下，Gossip 传播较慢可能导致检测时间更长（如节点数较多、网络拥塞等），使 failover 明显超出 node-timeout。
```

**解决**：
```bash
# 1. 适当减小 cluster-node-timeout（误判风险和恢复速度之间的权衡）
cluster-node-timeout 10000  # 10 秒，减少中断时间

# 2. 增加从节点数量
# 多个从节点可以提高选举成功率

# 3. 使用 Redis Sentinel（独立部署的故障检测）
# Sentinel 比 Cluster 自带的 Gossip 检测更可靠
```

---

## 七、云 Redis (DCS) 运维问题

### 案例1：带宽限制导致突发超时

**现象**：
- 突发流量时出现大量超时
- CPU/内存使用率正常，但 QPS 断崖式下降
- 控制台带宽监控显示近上限

**排查**：
```bash
# 查看云监控（非 Redis 自身指标）
# 华为云 DCS 控制台 → 监控 → 带宽使用率

# 检查 value 大小
redis-cli --bigkeys
```

**根因**：
```
云 Redis 每种规格有隐含的带宽上限：
  2GB 主备实例：最大带宽 48MB/s
  8GB 主备实例：最大带宽 192MB/s

超出带宽时，云平台软件层主动限速（throttle）：
  - 虚拟交换机/云防火墙/网络代理层设置带宽阈值
  - 超限则丢弃/延迟数据包
  - 自建 Redis 只受物理网卡限制，没有此问题

磁盘带宽也是类似限制（云平台虚拟化层配额）：
  - 持久化开启时，AOF 写入/RDB 快照需要落盘
  - 共享存储有 IOPS 带宽上限
  - 超出配额导致持久化延迟
```

**解决**：
```bash
# 1. 监控控制台带宽使用率（核心指标）

# 2. 避免热 key 大 value
# 单个 value 建议 < 10KB
# 大 value 压缩后存储（gzip/snappy）

# 3. 升级规格或切换集群实例
# 集群实例带宽按分片叠加

# 4. 控制台查看磁盘流量/IOPS 指标
# 重持久化场景选择高 IO 型实例
```

---

### 案例2：后台运维操作导致闪断

**现象**：
- 业务无任何变更，Redis 突然出现几秒到几十秒闪断
- 控制台显示"主备切换"或"HA 切换"事件

**排查**：
```bash
# 查看华为云 DCS 事件通知
# 控制台 → 实例详情 → 运维日志

# 客户端日志
# Connection reset / Read timeout
```

**根因**：
```
云平台不可控的后台运维操作：
  1. 热补丁升级：底层 Redis 内核打安全补丁，触发主备切换
  2. 宿主机迁移：硬件异常检测，热迁移到其他宿主机
  3. 主备倒换演练：部分云平台定期 HA 演练
  4. 证书轮换：SSL 加密传输时证书更新触发重连

这些操作在自建环境中完全可控，云上是被动承受的
```

**解决**：
```bash
# 1. 客户端实现自动重连 + 重试
# 建议重试 3 次，退避间隔 100ms/200ms/500ms

# 2. 使用支持自动重连的客户端
# Java: Lettuce（推荐）> Jedis（不支持自动重连）
# Go: go-redis（支持自动重连）

# 3. 关注运维公告和事件通知
# 设置告警，提前规避

# 4. 多可用区部署
# 降低单 AZ 故障影响

# 5. 客户端熔断/降级
# Redis 不可用时回源 DB
```

---

### 案例3：热 Key 导致数据倾斜

**现象**：
- 单分片 CPU 100%，其他分片空闲
- 华为云监控告警"存在热Key"或"分片不均衡"

**排查**：
```bash
# 使用华为云 DCS 热 Key 分析功能
# 控制台 → 诊断 → 热Key分析

# 手动观察
redis-cli --hotkeys  # Redis 4.0+ 支持
# 或 MONITOR 抓取实时流量
```

**根因**：
```
热 key 导致数据倾斜：

分片1: [slot 0-5460]     热key "product:1001" → CRC16 → slot 1234 → 分片1
分片2: [slot 5461-10922]  → 空闲
分片3: [slot 10923-16383] → 空闲

分片1 CPU 100%，分片2/3 CPU 10%
整体吞吐被单分片瓶颈限制

云上放大效应：
  单分片规格通常比自建小（如 2-8GB）
  更容易触发资源限制
```

**解决**：
```bash
# 热 Key 处理：
# 1. 本地缓存
# 在应用层缓存热 key（Caffeine/Guava）
# 减少对 Redis 的直接访问

# 2. key 分散
# product:1001 → product:1001:{随机后缀0-9}
# 分散到不同 slot 上

# 大 Key 处理：
# 3. 拆分
# 100 万 field 的 hash → 100 个 1 万的 hash

# 4. UNLINK 异步删除
redis-cli UNLINK big_key

# 5. 华为云 DCS 分析工具
# 控制台 → 诊断 → 热Key分析 / 大Key分析
```

---

### 案例4：网络隔离导致无法连接

**现象**：
- 应用无法连接 Redis
- Connection timeout
- 同 VPC 内可以连，跨 VPC 不行

**排查**：
```bash
# 1. 测试连通性
telnet <redis-ip> 6379

# 2. 确认网络位置
# 应用与 Redis 是否同一 VPC/子网

# 3. 检查安全组规则
# 是否放行应用所在子网的入站流量（6379 端口）

# 4. 跨 VPC 检查 Peering
```

**根因**：
```
云 Redis 部署在 VPC（虚拟私有云）内，受网络隔离约束：

  1. 安全组规则未放行 → 连接被防火墙拦截
  2. 跨 VPC 默认隔离 → 需 VPC Peering / 云连接
  3. 跨 Region 不可直接访问 → 华为云 DCS 不支持
  4. 公网默认不开放 → 需 ECS 跳板机
```

**解决**：
```bash
排查步骤：
1. 确认应用与 Redis 在同一 VPC 和子网
2. 检查安全组入站规则是否放行 6379 端口
3. 跨 VPC 时检查 Peering 连接和路由表
4. 使用 telnet <redis-ip> 6379 测试网络连通性
```

---

## 八、快速排查命令集合

```bash
# 基础
redis-cli INFO                          # 全局状态
redis-cli INFO server                   # 版本/进程信息
redis-cli INFO clients                  # 连接信息
redis-cli INFO memory                   # 内存信息
redis-cli INFO stats                    # 统计信息（命中率、fork耗时等）
redis-cli INFO persistence              # 持久化状态
redis-cli INFO replication              # 主从复制状态
redis-cli INFO cpu                      # CPU 使用

# 慢查询
redis-cli SLOWLOG GET 10
redis-cli SLOWLOG LEN
redis-cli CONFIG SET slowlog-log-slower-than 10000

# 延迟
redis-cli --latency -h <host> -p <port>
redis-cli --latency-history -i 1

# 大 Key
redis-cli --bigkeys

# 内存分析
redis-cli MEMORY STATS
redis-cli MEMORY DOCTOR
redis-cli MEMORY USAGE <key>
redis-cli MEMORY PURGE

# 集群
redis-cli CLUSTER INFO
redis-cli CLUSTER NODES
redis-cli CLUSTER SLOTS

# 排查 Swap
cat /proc/`pidof redis-server`/status | grep Swap

# 排查 THP
cat /sys/kernel/mm/transparent_hugepage/enabled

# 排查 ulimit
cat /proc/`pidof redis-server`/limits | grep "open files"
```

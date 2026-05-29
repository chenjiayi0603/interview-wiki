# Redis 持久化问题

See also: [[内存管理]]

## 现象
- 持久化失败
- 数据丢失

## 可能原因
- 磁盘空间不足，RDB/AOF 写入失败
- Redis 进程对 `dir` 目录无写权限
- 仅用 RDB：宕机时未到下次 save 点，丢失最后一次快照后的数据
- AOF 未开启或 fsync 策略过于宽松（如 `everysec`/`no`），宕机丢缓冲区内数据
- RDB 或 AOF 文件损坏（异常断电、磁盘坏道、被误删改），重启无法加载
- fork 失败（如内存不足）导致 bgsave/bgrewriteaof 无法启动

## 排查步骤
1. **RDB 快照状态**：检查 `lastsave`、`rdb_last_save_time` 及 `rdb_bgsave_in_progress`
2. **AOF 重写状态**：`aof_rewrite_in_progress`、`aof_current_size`
3. **磁盘空间与权限**：确认 dir 目录空间充足且 Redis 有写权限

## 解决方案
- 检查磁盘空间与目录权限
- 优化持久化触发策略（`save` 规则、`auto-aof-rewrite-percentage` 等）
- 使用混合持久化（RDB + AOF）提升恢复效率

## 根因原理深入分析

### RDB vs AOF 的本质区别与数据丢失原理

**RDB（快照）**：某一时刻的全量内存数据二进制序列化
```
时间线：[save点1] -------- 写入100条 -------- [宕机]
                                                ↑ 丢失这100条
```
RDB 是"定时拍照"，两次快照之间的数据必然有丢失风险。save 间隔越大，丢失窗口越大。

**AOF（追加日志）**：每个写命令追加到文件末尾
```
命令执行 → 写入 AOF 缓冲区 → fsync 到磁盘

fsync 策略：
- always：每个命令都 fsync → 最安全但最慢（性能下降约 50%+）
- everysec：每秒 fsync 一次 → 最多丢 1 秒数据（生产推荐）
- no：由 OS 决定何时 flush → 可能丢失 30 秒数据
```

**AOF 重写（BGREWRITEAOF）的原理**：
AOF 文件会持续增长（记录每一个写命令），需要定期重写压缩：
1. fork 子进程，对当前内存数据生成最小命令集（如 1000 次 INCR 合并为一次 SET）
2. 父进程新写入同时写到 AOF 缓冲区 + AOF 重写缓冲区
3. 子进程完成后，父进程将重写缓冲区的增量追加到新 AOF 文件
4. 原子替换旧 AOF 文件

**AOF 损坏的处理**：
```bash
# 检查 AOF 文件完整性
redis-check-aof --fix appendonly.aof
# 会截断到最后一个完整的命令，丢弃损坏的尾部
```

### 混合持久化（Redis 4.0+）的原理
```
AOF 重写时：RDB格式（内存快照） + AOF增量（重写期间新命令）
  ┌────────────────┬──────────────┐
  │  RDB 二进制数据  │  AOF 命令日志  │
  └────────────────┴──────────────┘
  恢复时先加载 RDB 部分（快），再回放 AOF 部分（完整）
```
兼顾了 RDB 的加载速度和 AOF 的数据完整性，是当前生产环境的推荐方案。

```bash
# 开启混合持久化
aof-use-rdb-preamble yes  # Redis 4.0+，5.0+ 默认开启
```

[src: raw/ingested/2技术/redis/Redis_KeyDB运维问题速查.md]

## Related Pages
- [[内存管理]]
- [[Redis性能问题]]
- [[Redis内存问题]]
- [[Redis数据不一致]]
- [[Redis可观测性项目]]
- [[性能优化]]

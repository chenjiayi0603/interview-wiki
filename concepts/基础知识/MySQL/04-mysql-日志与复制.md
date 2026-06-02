# MySQL 日志与复制

> Redo Log、Binlog、Undo Log、两阶段提交（2PC）、组提交、主从复制、半同步/增强半同步、并行复制。

---

## 一、三大日志对比

| 维度 | Redo Log | Binlog | Undo Log |
|------|---------|--------|---------|
| **所属层** | InnoDB 引擎层 | MySQL Server 层 | InnoDB 引擎层 |
| **记录内容** | 物理日志：页的修改 | 逻辑日志：SQL/行变化 | 修改前的数据 |
| **写入方式** | 循环写入（固定大小） | 追加写入（不断增长） | 追加写入 |
| **主要用途** | 崩溃恢复（持久性） | 主从复制、数据恢复 | 回滚（原子性）+ MVCC |
| **刷盘控制** | `innodb_flush_log_at_trx_commit` | `sync_binlog` | 自动 |

### 1.1 Redo Log（重做日志）

**作用：** 保证 ACID 的持久性（Durability），实现 WAL（Write-Ahead Logging）。

```sql
-- 刷盘策略
innodb_flush_log_at_trx_commit = 1  -- 每次提交刷盘（最安全）
                                  = 2  -- 每次提交写 OS 缓存
                                  = 0  -- 每秒刷一次
```

**Redo Log 格式：** 记录「哪个页的哪个偏移量，写入什么内容」，而不是记录了哪个字段。

### 1.2 Binlog（二进制日志）

**作用：** 主从复制、数据恢复、数据审计。

```sql
-- Binlog 三种格式
binlog_format = STATEMENT  -- SQL 语句（日志量小，但可能主从不一致）
              = ROW        -- 行数据变化（精确，推荐）
              = MIXED      -- 混合
```

### 1.3 Undo Log（回滚日志）

**作用：** 事务回滚（原子性）+ MVCC 版本链构建。

**关键点：** Undo Log 的修改本身也会产生 Redo Log（Undo 页的修改受 Redo 保护）。

---

## 二、两阶段提交（2PC）

### 2.1 为什么需要 2PC？

Redo Log（引擎层）和 Binlog（Server 层）是**两个独立系统**，必须保证一致性。

**没有 2PC 的问题：**

```
先写 Redo 后写 Binlog：
  T1: Redo Prepare ✓
  T2: 写 Binlog ...
  ─── 宕机 ───
 → Redo 恢复了数据，但 Binlog 没有 → 从库丢数据

先写 Binlog 后写 Redo：
  T1: Binlog 写入 ✓
  T2: Redo Log ...
  ─── 宕机 ───
 → Binlog 有数据，但 Redo 没恢复 → 主库丢数据
```

### 2.2 2PC 流程

```
Phase 1（Prepare）：
  Redo Log 刷盘，标记为 TRX_PREPARED

Phase 2（Commit）：
  ① 写 Binlog（含 XID event），刷盘
  ② Redo Log 写入 Commit 标记
```

### 2.3 崩溃恢复规则

```
扫描 Redo Log 中所有 PREPARED 状态的事务
   ├── Binlog 中有对应 XID → 提交事务
   └── Binlog 中无对应 XID → 回滚事务

铁律：以 Binlog 为准！
  Binlog 有 → 主库必须提交（保证主从一致）
  Binlog 无 → 主库必须回滚
```

### 2.4 四种故障场景

| 场景 | 状态 | 恢复结果 | 一致性 |
|------|------|---------|:------:|
| Redo Prepare 前宕机 | 事务未持久化 | 回滚 | ✓ |
| Redo Prepare 后、写 Binlog 前宕机 | Binlog 无 XID | 回滚 | ✓ |
| Binlog 写入后、Redo Commit 前宕机 | Binlog 有 XID | 提交 | ✓ |
| Redo Commit 后宕机 | 事务已提交 | 无需处理 | ✓ |

---

## 三、组提交（Group Commit）

### 3.1 原理

将多个并发事务的刷盘操作合并为一次，减少 fsync 次数。

```
无组提交：3 个事务 = 6 次 fsync
有组提交：3 个事务 = 2 次 fsync（1 次 Redo + 1 次 Binlog）
```

### 3.2 三阶段队列

MySQL 5.7+ 将组提交优化为三个独立的流水线阶段：

```
事务1 ─┐
事务2 ─┼──► [Flush 阶段] ──► [Sync 阶段] ──► [Commit 阶段]
事务3 ─┘        │                  │                │
           Redo Log 刷盘      Binlog 刷盘      Redo Commit
           (Prepare)          (Sync)           (引擎层提交)
```

### 3.3 相关参数

```sql
binlog_group_commit_sync_delay = 100          -- 微秒，等待更多事务
binlog_group_commit_sync_no_delay_count = 10  -- 积累 N 个事务后不等
```

---

## 四、主从复制

### 4.1 复制流程

```
主库 (Master)                       从库 (Slave)
  │                                    │
  ① 事务提交，写 Binlog                 │
  ② Dump Thread 推送 Binlog  ──────►  ③ I/O Thread 接收
                                         写入 Relay Log
                                       ④ SQL Thread 重放
                                         写入 Slave DB
```

**三个线程：**

| 线程 | 位置 | 功能 |
|------|------|------|
| **Dump Thread** | 主库 | 每从库一个，推送 Binlog |
| **I/O Thread** | 从库 | 拉取 Binlog，写入 Relay Log |
| **SQL Thread** | 从库 | 读取 Relay Log 并执行 |

### 4.2 三种复制模式

| 模式 | ACK 等待点 | 数据丢失风险 | 性能 |
|------|-----------|:-----------:|:----:|
| **异步复制** | 不等待 | 可能大量丢失 | 最佳 |
| **传统半同步（AFTER_COMMIT）** | Redo Commit 之后等 | 可能丢少量已提交数据 | 中等 |
| **增强半同步/无损（AFTER_SYNC）** | Redo Commit 之前等 | 几乎为零 | 中等 |

### 4.3 增强半同步（无损复制）流程

```
AFTER_SYNC 模式（MySQL 5.7.2+）：
  ① Redo Prepare
  ② 写 Binlog 并刷盘
  ③ 发送 Binlog 给从库 ← 关键：引擎层尚未提交！
  ④ 等待从库 ACK
  ⑤ 收到 ACK → Redo Commit（引擎层提交）
  ⑥ 返回客户端成功

参数配置：
  rpl_semi_sync_master_wait_point = AFTER_SYNC
  rpl_semi_sync_master_timeout = 10000  -- ms，超时降级为异步
```

### 4.4 半同步配置

```sql
-- 主库
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_wait_point = AFTER_SYNC;

-- 从库
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

```ini
# my.cnf
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1
gtid_mode = ON
enforce_gtid_consistency = ON

# 半同步
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_master_wait_point = AFTER_SYNC
```

### 4.5 并行复制

让从库 SQL 线程多线程并行重放 Relay Log，降低主从延迟。

```sql
STOP SLAVE SQL_THREAD;
SET GLOBAL slave_parallel_workers = 8;           -- Worker 数（4~16）
SET GLOBAL slave_parallel_type = LOGICAL_CLOCK;  -- 基于组提交
SET GLOBAL slave_preserve_commit_order = ON;     -- 保持提交顺序
START SLAVE SQL_THREAD;
```

### 4.6 监控命令

```sql
-- 主库
SHOW MASTER STATUS;                       -- Binlog 位置
SHOW STATUS LIKE 'Rpl_semi_sync%';        -- 半同步状态

-- 从库
SHOW SLAVE STATUS\G                       -- 复制状态、延迟
-- 关注: Slave_IO_Running, Slave_SQL_Running, Seconds_Behind_Master
```

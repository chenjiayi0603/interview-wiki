# MySQL 读写流程面试高频问题

> 本文档收录 MySQL 读写流程相关的面试高频问题，涵盖 UPDATE 执行过程、Buffer Pool LRU 淘汰策略、Redo Log 必要性、Change Buffer 机制、事务提交后数据持久化、读写流程中的锁等核心知识点。

---

## Q1: 描述一条 UPDATE 语句在 MySQL 中的完整执行过程

```
以 UPDATE users SET age = 26 WHERE id = 100 为例：

 1. 连接器：接收连接，验证身份权限
 2. 分析器：词法语法分析，生成 AST
 3. 优化器：选择执行计划（主键 id=100 → 主键索引，const 级别）
 4. 执行器：调用 InnoDB Handler API 执行更新

【进入 InnoDB 引擎层】
 5. 写 Undo Log：记录 age 修改前的值（25），用于回滚和 MVCC
 6. 修改 Buffer Pool：在内存中修改数据页，age 25→26，标记脏页
 7. 写 Redo Log Buffer：记录页修改的物理日志

【提交时：两阶段提交】
 8. Redo Prepare：Redo Log Buffer fsync 到 ib_logfile，标记 PREPARED
 9. 写 Binlog：Binlog Cache fsync 到 mysql-bin.xxx
10. Redo Commit：Redo Log 写入 Commit 标记

【返回+后台】
11. 返回客户端 "1 row affected"
12. 后台 Page Cleaner 将脏页异步写入 .ibd 数据文件
```

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

## Q2: Buffer Pool 的 LRU 淘汰策略是怎样的？

```
InnoDB 使用改进的 LRU 算法（Midpoint Insertion）：

┌──────────────────────────┬────────────────────────┐
│     Young (热端) 5/8      │     Old (冷端) 3/8      │
│                           │                        │
│  经常访问 → 移到头部      │  新页插入 → 插入头部    │
│                           │  淘汰 ← 从尾部移除      │
└──────────────────────────┴────────────────────────┘

 防止全表扫描冲掉热数据：
   - 新页加载时插入 Old 区域头部（不是 Young 头部）
   - 在 Old 区域等待 innodb_old_blocks_time (默认1000ms)
   - 只有在此期间被再次访问的页才晋升到 Young 区域
   - 一次性的全表扫描页会在 Old 区域被快速淘汰
```

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

## Q3: 为什么需要 Redo Log？直接刷脏页不行吗？

```
两个原因：

① 性能（核心原因）：
   - 数据页刷盘是随机写（不同页在不同磁盘位置）
   - Redo Log 是顺序写（追加写，无寻道开销）
   - 顺序写的速度是随机写的几十甚至上百倍
   - 先将修改记录在 Redo Log（快），再慢慢刷脏页（慢）

② 崩溃恢复：
   - 如果刷脏页过程中崩溃，部分数据页可能是"半完成"状态
   - Redo Log 记录了完整的修改，可以精确重放
   - 这就是 WAL (Write-Ahead Logging) 的核心思想

所以策略是：
  事务提交时：fsync Redo Log (顺序写，快) → 返回成功
  后台空闲时：慢慢刷脏页 (随机写，慢) → 持久化到主数据文件
```

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

## Q4: Change Buffer 是什么？什么场景下有效？

```
Change Buffer 是 InnoDB 的一种写优化机制：

  当修改的二级索引页不在 Buffer Pool 中时，
  不立即从磁盘加载该页，而是把修改操作记录在 Change Buffer 中。

  本质：将"随机读+修改+写回"变为"纯顺序写 + 延迟Merge"

适用条件（极其严格）：
  ① 必须是【二级索引】（非聚簇索引）
  ② 索引页【不在 Buffer Pool 中】
  ③ 不能是【唯一索引】（唯一索引需要读盘验证唯一性）

最适用场景：写多读少
  - 例如：日志表、流水表、账单表 → 插入大量数据但很少查询
  - 例如：批量数据导入 → 二级索引的写入被 Change Buffer 吸收

不适用场景：读多写少
  - Change Buffer 中的修改需要在读取时 Merge，可能反而变慢
```

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

## Q5: 事务提交后，数据是否立即写入磁盘？

```
答案：不一定！

  【立刻写入磁盘的】
  ✓ Redo Log：innodb_flush_log_at_trx_commit=1 时 fsync 到 ib_logfile
  ✓ Binlog：sync_binlog=1 时 fsync 到 mysql-bin.xxx

  【不立刻写入磁盘的】
  ✗ 数据文件 (.ibd)：脏页由 Page Cleaner 后台线程异步刷盘
  ✗ 数据文件可能很久以后才写入

  关键理解：
  ┌──────────────────────────────────────────────────────────┐
  │ 事务的持久性（Durability）由 Redo Log 保证，不是由数据文件！ │
  │                                                          │
  │ 崩溃恢复流程：                                             │
  │   Redo Log（有Prepare+Commit） → 重放在Buffer Pool中的修改 │
  │      → 恢复已提交但未刷盘的数据                             │
  │   Undo Log → 回滚未提交的事务                             │
  │                                                          │
  │ 即使 .ibd 文件是旧版本，Redo Log 也能将其恢复到最新状态     │
  └──────────────────────────────────────────────────────────┘
```

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

## Q6: 读写流程中有哪些锁？

```
读流程使用的锁：
  - 普通 SELECT（快照读）：不加锁，通过 MVCC 读取历史版本
  - SELECT ... FOR UPDATE（当前读/锁定读）：加 Next-Key Lock
  - SELECT ... LOCK IN SHARE MODE：加共享 Next-Key Lock

  隔离级别影响：
    RC 下：只对命中的行加 Record Lock
    RR 下：加 Next-Key Lock (Record Lock + Gap Lock)

写流程使用的锁：
  - INSERT：加插入意向锁 (Insert Intention Lock)
  - UPDATE/DELETE：先在二级索引加锁，再回主键索引加锁
    · 如果修改了索引列，涉及旧索引的删除和新索引的插入

  MDL (Metadata Lock)：
    DML 执行期间持有 MDL_SHARED_READ
    DDL 需要 MDL_EXCLUSIVE，会阻塞所有 DML
```

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

---

*本文档系统梳理了 MySQL 的读流程和写流程，涵盖 Server 层解析与 InnoDB 存储引擎的内部协作机制，适用于面试准备与性能调优实践。*

[src: raw/ingested/2技术/mysql/MySQL读写流程详解-五、读写流程面试高频问题.md]

## Related Pages
- [[MySQL事务提交与两阶段提交]]
- [[MySQL写入流程DML修改]]
- [[MySQL架构与存储引擎]]
- [[MySQL索引]]
- [[MySQL事务与锁]]
- [[Checkpoint机制]]
- [[数据库性能优化要点]]
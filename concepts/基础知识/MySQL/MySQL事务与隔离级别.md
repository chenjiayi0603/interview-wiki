# MySQL事务与隔离级别

See also: [[MySQL存储引擎]]

## ACID 事务四大基本特性

- **A（原子性，Atomicity）**：事务中的操作要么全部执行，要么全部不执行，不会出现部分执行的中间状态。
- **C（一致性，Consistency）**：事务执行前后，数据库都处于一致性状态（如满足唯一性、主外键等约束）。
- **I（隔离性，Isolation）**：多个并发事务之间互不干扰，各自"像独占数据库一样执行"。
- **D（持久性，Durability）**：事务提交后，对数据库的修改会永久保存，即使系统故障也不丢失。

## ACID 事务特性的实现

- **原子性**：undo log（回滚日志）
- **持久性**：redo log（重做日志，WAL机制）
- **隔离性**：锁机制 + MVCC
- **一致性**：通过前三者共同保障

## 事务隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 | 实现机制 |
|---------|------|-----------|------|------|---------|
| **READ UNCOMMITTED** | ❌ | ❌ | ❌ | 最高 | 无锁 |
| **READ COMMITTED** | ✅ | ❌ | ❌ | 较高 | MVCC（每次查询新快照） |
| **REPEATABLE READ** | ✅ | ✅ | ⚠️ | 中等 | MVCC（一致性快照）+ 间隙锁 |
| **SERIALIZABLE** | ✅ | ✅ | ✅ | 最低 | 表级锁 |

> **注**：InnoDB引擎在RR级别通过间隙锁（Gap Lock）和临键锁（Next-Key Lock）解决幻读，但普通查询仍可能因"当前读"出现幻读。

## 并发问题详解

| 问题类型 | 含义 | 严重程度 |
|---------|------|---------|
| **脏读 (Dirty Read)** | 读到其他事务未提交的数据 | ⚠️ 严重 |
| **不可重复读 (Non-repeatable Read)** | 同一事务两次读同一行数据结果不同 | ⚠️ 中等 |
| **幻读 (Phantom Read)** | 同一事务两次查询条件范围数据结果数量不同 | ⚠️ 中等 |
| **丢失更新 (Lost Update)** | 多个事务同时更新同一数据，后提交的覆盖先提交的 | ⚠️ 严重 |

### 丢失更新解决方案
1. **悲观锁**：使用 `SELECT ... FOR UPDATE` 加排他锁
2. **乐观锁**：使用版本号或时间戳
3. **原子操作**：直接使用 SQL 的原子更新（如 `UPDATE account SET balance = balance + 100 WHERE id = 1;`）

## 事务控制语句

```sql
-- 开始事务
START TRANSACTION;  -- 或 BEGIN

-- 提交事务
COMMIT;

-- 回滚事务
ROLLBACK;

-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置全局隔离级别
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 设置当前会话隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

[src: raw/ingested/2技术/mysql/mysql复习文档.md]

## Related Pages
- [[MVCC多版本并发控制]]
- [[MySQL存储引擎]]
- [[MySQL锁机制]]
- [[MySQL性能优化]]
- [[MySQL分库分表]]
- [[MySQL主从复制与高可用]]

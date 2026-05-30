# MySQL事务与锁

> 本文档整合了事务四大基本特性（ACID）、MVCC 多版本并发控制、锁机制速览、存储引擎对比等核心内容。

## 一、事务四大基本特性（ACID）

- **A（原子性，Atomicity）**：事务中的操作要么全部执行，要么全部不执行，不会出现部分执行的中间状态。
- **C（一致性，Consistency）**：事务执行前后，数据库都处于一致性状态（如满足唯一性、主外键等约束）。
- **I（隔离性，Isolation）**：多个并发事务之间互不干扰，各自"像独占数据库一样执行"。常见隔离级别有RC、RR、Serializable等。
- **D（持久性，Durability）**：事务提交后，对数据库的修改会永久保存，即使系统故障也不丢失。

### 事务特性的实现

- **原子性**：undo log（回滚日志）
- **持久性**：redo log（重做日志，WAL机制）
- **隔离性**：锁机制 + MVCC
- **一致性**：通过前三者共同保障

### 隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 | 实现机制 |
|---------|------|-----------|------|------|---------|
| **READ UNCOMMITTED** | ❌ | ❌ | ❌ | 最高 | 无锁 |
| **READ COMMITTED** | ✅ | ❌ | ❌ | 较高 | MVCC（每次查询新快照） |
| **REPEATABLE READ** | ✅ | ✅ | ⚠️ | 中等 | MVCC（一致性快照）+ 间隙锁 |
| **SERIALIZABLE** | ✅ | ✅ | ✅ | 最低 | 表级锁 |

> MySQL InnoDB 默认隔离级别是 REPEATABLE READ，通过 Next-Key Lock 基本解决了幻读问题。

## 二、MVCC 多版本并发控制

| 组件 | 作用 |
|------|------|
| **隐藏列** | DB_TRX_ID（事务ID）、DB_ROLL_PTR（回滚指针）、DB_ROW_ID（行ID） |
| **undo log** | 存储历史版本，形成版本链 |
| **ReadView** | 事务快照，判断数据可见性（low_limit_id、up_limit_id、rw_trx_ids） |

**隔离级别差异**：
- **RC**：每次查询生成新ReadView
- **RR**：事务开始时生成ReadView，整个事务复用

## 三、锁机制速览

| 锁类型 | 粒度 | 说明 |
|-------|------|------|
| **表级锁** | 整表 | MyISAM默认，并发度低 |
| **行级锁** | 单行 | InnoDB默认，并发度高 |
| **Record Lock** | 记录锁 | 锁定单行记录 |
| **Gap Lock** | 间隙锁 | 锁定索引记录间隙，防止幻读 |
| **Next-Key Lock** | 临键锁 | Record + Gap，InnoDB默认行锁实现 |

**死锁4要素**：互斥、请求与保持、不剥夺、循环等待

## 四、存储引擎对比

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **索引** | 聚簇索引（主键）+ 二级索引 | 非聚簇索引（索引与数据分离） |
| **事务** | ✅ 支持 | ❌ 不支持 |
| **锁** | 行级锁 | 表级锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **MVCC** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 支持 | ❌ 不支持 |
| **适用场景** | 高并发写、事务要求 | 读多写少、日志分析 |

**外键（Foreign Key）**：约束子表字段只能引用主表已有主键，保障表间数据一致性，防止"孤儿"数据。

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    amount DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

[src: raw/ingested/2技术/mysql/数据库基础概念与范式设计.md]

## Related Pages
- [[MySQL死锁]]
- [[MySQL事务提交与两阶段提交]]
- [[MySQL索引]]
- [[MySQL架构与存储引擎]]
- [[数据库语言分类]]

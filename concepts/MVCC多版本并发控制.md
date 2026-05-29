# MVCC多版本并发控制

See also: [[MySQL性能优化]]

## 概述

**MVCC（Multi-Version Concurrency Control）** 是MySQL实现非锁定读的核心机制，主要适用于MySQL的RC（Read Committed）和RR（Repeatable Read）隔离级别。

**核心思想**：
- 通过保存数据在某个时间点的快照来实现
- 一个事务无论运行多长时间，在同一个事务里能够看到数据一致的视图
- 根据事务开始的时间不同，在同一个时刻不同事务看到的相同表里的数据可能是不同的

## 实现原理

### 隐藏列
InnoDB为每一行数据增加了三个隐藏列用于实现MVCC：

| 列名 | 长度(字节) | 作用 |
|-----|----------|------|
| **DB_TRX_ID** | 6 | 插入或更新行的最后一个事务的事务标识符（删除视为更新，将其标记为已删除） |
| **DB_ROLL_PTR** | 7 | 回滚指针，指向undo log中该行的历史版本 |
| **DB_ROW_ID** | 6 | 行ID（如果没有主键，InnoDB会自动生成） |

### undo log版本链
- 每次对数据进行修改时，都会在undo log中记录修改前的数据版本
- 通过DB_ROLL_PTR指针，可以找到该行的所有历史版本，形成一个版本链

### ReadView
ReadView是事务在某一时刻给整个事务系统（trx_sys）打的快照，用于判断数据对该事务是否可见。

**ReadView的主要内容**：
- **low_limit_id**：表示生成ReadView时系统中应该分配给下一个事务的id。如果数据的事务id大于等于low_limit_id，则对该ReadView不可见。
- **up_limit_id**：表示生成ReadView时当前系统中活跃的读写事务中最小的事务id。如果数据的事务id小于up_limit_id，则对该ReadView可见。
- **rw_trx_ids**：表示生成ReadView时当前系统中活跃的读写事务的事务id列表。

## 不同隔离级别的MVCC实现

### Read Committed (RC)
- 每次查询都会生成一个新的ReadView
- 因此每次查询都能看到最新已提交的数据

### Repeatable Read (RR)
- 事务开始时生成一个ReadView，整个事务期间都使用这个ReadView
- 因此同一事务内的多次查询结果一致

[src: raw/ingested/2技术/mysql/mysql复习文档.md]

## Related Pages
- [[MySQL事务与隔离级别]]
- [[MySQL存储引擎]]
- [[MySQL锁机制]]
- [[MySQL索引]]
- [[MySQL性能优化]]

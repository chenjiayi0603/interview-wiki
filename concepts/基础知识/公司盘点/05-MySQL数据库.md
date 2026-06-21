# MySQL 速查

> 中小公司必问

---

## 一、B+Tree 索引

```
聚簇索引: 主键即数据（主键查询优先）
二级索引: 存主键值，回表查数据
联合索引: 最左前缀原则
Hash 索引: 等值查询 O(1)，不支持范围
```

---

## 二、事务隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|:---:|:---------:|:---:|
| READ UNCOMMITTED | ❌ | ❌ | ❌ |
| READ COMMITTED | ✅ | ❌ | ❌ |
| REPEATABLE READ | ✅ | ✅ | ❌（MVCC） |
| SERIALIZABLE | ✅ | ✅ | ✅ |

---

## 三、MVCC

```
每行隐藏字段: DB_TRX_ID + DB_ROLL_PTR（undo log 链）

Read View:
  trx_id < min_trx_id → 可见
  trx_id > max_trx_id → 不可见
  trx_id in m_ids → 不可见

RR: Read View 不变（可重复读）
RC: 每次查询新建 Read View
```

---

## 四、索引失效场景

```
LIKE '%xxx' / 函数包裹 / 隐式类型转换
不满足最左前缀 / OR 含非索引列 / != NOT IN
```

---

## 五、锁

```
行锁: InnoDB 默认
间隙锁(Gap Lock): RR 级别防幻读
临键锁(Next-Key): 行锁+间隙锁
表锁: DDL 操作
```

# MySQL 事务与锁

> ACID、隔离级别、MVCC、锁类型（Record/Gap/Next-Key）、死锁。

---

## 一、ACID 特性

| 特性 | 含义 | 实现机制 |
|------|------|---------|
| **原子性（Atomicity）** | 事务内操作要么全成功，要么全回滚 | **undo log** |
| **一致性（Consistency）** | 事务前后数据满足所有约束 | 原子性+持久性+隔离性共同保障 |
| **隔离性（Isolation）** | 并发事务互不干扰 | **锁 + MVCC** |
| **持久性（Durability）** | 提交后修改永久保存 | **redo log（WAL）** |

---

## 二、事务隔离级别

### 2.1 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现机制 |
|---------|:----:|:---------:|:----:|---------|
| **READ UNCOMMITTED** | ❌ | ❌ | ❌ | 无锁，读最新数据 |
| **READ COMMITTED** | ✅ | ❌ | ❌ | MVCC（每次查询生成新 ReadView） |
| **REPEATABLE READ**（默认） | ✅ | ✅ | ✅¹ | MVCC（事务内同一 ReadView）+ 间隙锁 |
| **SERIALIZABLE** | ✅ | ✅ | ✅ | 表级锁 |

¹ InnoDB 在 RR 级别通过 Gap Lock 和 Next-Key Lock 解决幻读。

### 2.2 并发问题示例

**脏读：读到未提交的数据**

```sql
-- T1 更新未提交
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 未提交

-- T2 读到未提交的数据（脏读）
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT balance FROM account WHERE id = 1;  -- 读到 900

-- T1 回滚
ROLLBACK;  -- balance 变回 1000，T2 之前读到的 900 是脏数据
```

**不可重复读：同一事务两次读同一行结果不同**

```sql
-- T1 第一次读
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;  -- 1000

-- T2 修改并提交
UPDATE account SET balance = 1500 WHERE id = 1;
COMMIT;

-- T1 第二次读（结果变了）
SELECT balance FROM account WHERE id = 1;  -- 1500
-- 同一事务内两次读取结果不一致！
```

**幻读：同一事务两次范围查询行数不同**

```sql
START TRANSACTION;
SELECT COUNT(*) FROM account WHERE balance > 1000;  -- 5

-- 其他事务插入一条 balance=2000 的数据并提交

SELECT COUNT(*) FROM account WHERE balance > 1000;  -- 6（多了一条）
```

### 2.3 隔离级别设置

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置会话级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局级别
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

## 三、MVCC（多版本并发控制）

### 3.1 核心组件

| 组件 | 作用 |
|------|------|
| **DB_TRX_ID** | 隐藏列，记录最后修改该行的事务 ID |
| **DB_ROLL_PTR** | 隐藏列，回滚指针，指向 undo log 中的历史版本 |
| **undo log** | 存储数据的历史版本，形成版本链 |
| **ReadView** | 事务快照，判断数据可见性 |

### 3.2 ReadView 判断规则

```
ReadView 包含：
- up_limit_id:  活跃事务中最小的事务 ID
- low_limit_id: 下一个待分配的事务 ID
- rw_trx_ids:   活跃事务 ID 列表

可见性判断：
  数据的事务 ID < up_limit_id       → 可见（已提交）
  数据的事务 ID >= low_limit_id     → 不可见（未来事务）
  数据的事务 ID 在 rw_trx_ids 中    → 不可见（活跃事务）
  数据的事务 ID 不在 rw_trx_ids 中  → 可见（已提交）
```

### 3.3 RC vs RR 的 MVCC 差异

- **RC**：每次 SELECT 生成新的 ReadView → 能读到其他事务已提交的修改
- **RR**：事务开始时生成 ReadView，整个事务复用 → 多次读结果一致

### 3.4 版本链示例

```
原始行：id=100, name='Tom', age=25

事务A UPDATE age=26：
  当前版本 (age=26, trx_id=100, roll_ptr →)
  Undo版本 (age=25, trx_id=90, roll_ptr →)
  Undo版本 (age=24, trx_id=80, roll_ptr → null)

通过 roll_ptr 串联版本链，MVCC 根据 ReadView 选择可见版本
```

---

## 四、锁机制

### 4.1 锁分类

**按粒度：**

| 锁类型 | 粒度 | 并发度 | 适用 |
|-------|------|--------|------|
| **表级锁** | 整表 | 低 | MyISAM |
| **行级锁** | 单行 | 高 | InnoDB |
| **页级锁** | 数据页 | 中 | 较少使用 |

**按类型：**

| 锁类型 | 兼容性 |
|-------|--------|
| **共享锁（S 锁）** | 多事务可同时持有 |
| **排他锁（X 锁）** | 与任何锁都不兼容 |
| **意向共享锁（IS）** | 表级标记，表示事务要加 S 锁 |
| **意向排他锁（IX）** | 表级标记，表示事务要加 X 锁 |

### 4.2 InnoDB 行锁三种实现

```sql
-- Record Lock（记录锁）：锁定单行
SELECT * FROM users WHERE id = 1 FOR UPDATE;

-- Gap Lock（间隙锁）：锁定记录之间的间隙，防止插入
-- 在 REPEATABLE READ 级别自动使用
SELECT * FROM users WHERE id BETWEEN 10 AND 20 FOR UPDATE;
-- 锁定 (10,20) 之间的间隙，防止其他事务插入 id=15 的记录

-- Next-Key Lock（临键锁）：Record Lock + Gap Lock 组合
-- InnoDB RR 级别默认的行锁实现
```

**锁定范围示例（索引值：1, 4, 7, 10）：**

```
Next-Key Lock 覆盖范围：
  (-∞, 1], (1, 4], (4, 7], (7, 10], (10, +∞)

SELECT * FROM t WHERE id = 5 FOR UPDATE;
-- 锁定 (4, 7] 范围，防止插入 id=5 或 id=6
```

### 4.3 加锁 SQL

```sql
-- 快照读（不加锁）
SELECT * FROM users WHERE id = 1;

-- 当前读（加 Next-Key Lock）
SELECT * FROM users WHERE id = 1 FOR UPDATE;
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;
UPDATE users SET name = 'Tom' WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

---

## 五、死锁

### 5.1 死锁 4 要素

| 要素 | 示例 |
|------|------|
| **互斥** | 行锁一次只能被一个事务持有 |
| **请求与保持** | 事务 A 持有锁 1，请求锁 2 |
| **不剥夺** | 锁只能由持有者主动释放 |
| **循环等待** | A 等 B，B 等 A |

### 5.2 死锁示例

```sql
-- 事务A
START TRANSACTION;
UPDATE users SET name = 'A' WHERE id = 1;  -- 持有 id=1 锁
UPDATE users SET name = 'A' WHERE id = 2;  -- 请求 id=2 锁（被 B 持有）

-- 事务B
START TRANSACTION;
UPDATE users SET name = 'B' WHERE id = 2;  -- 持有 id=2 锁
UPDATE users SET name = 'B' WHERE id = 1;  -- 请求 id=1 锁（被 A 持有）

-- 结果：循环等待 → 死锁
-- InnoDB 自动检测，回滚其中一个事务
```

### 5.3 死锁预防

1. **按相同顺序访问资源**：所有事务按 id 升序更新
2. **减少事务持有锁的时间**：大事务拆小
3. **使用较低的隔离级别**：RC 比 RR 死锁概率低
4. **为表添加合理索引**：减少锁冲突范围

```sql
-- 查看死锁信息
SHOW ENGINE INNODB STATUS\G
-- 关注 LATEST DETECTED DEADLOCK 部分
```

### 5.4 死锁调试案例

#### 案例：订单与库存表死锁

**表结构：**

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    product_id INT,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    INDEX idx_user (user_id),
    INDEX idx_product (product_id)
) ENGINE=InnoDB;

CREATE TABLE inventory (
    product_id INT PRIMARY KEY,
    stock INT,
    version INT
) ENGINE=InnoDB;
```

**死锁复现：**

```sql
-- 事务A（下单）
START TRANSACTION;
-- 步骤1：扣减库存（锁定 inventory product_id=100）
UPDATE inventory SET stock = stock - 1 WHERE product_id = 100;
-- 步骤2：创建订单（锁定 orders 的 idx_product 间隙）
INSERT INTO orders (id, user_id, product_id, amount, status)
VALUES (1001, 1, 100, 99.9, 'pending');

-- 事务B（查询订单状态并补库存）
START TRANSACTION;
-- 步骤1：查询订单（锁定 orders 的 idx_product 间隙）
SELECT * FROM orders WHERE product_id = 100 FOR UPDATE;
-- 步骤2：回补库存（锁定 inventory product_id=100）
UPDATE inventory SET stock = stock + 1 WHERE product_id = 100;

-- 结果：事务A 持有 inventory 锁，请求 orders 锁
--      事务B 持有 orders 锁，请求 inventory 锁
--      → 循环等待 → 死锁
```

#### 死锁诊断：SHOW ENGINE INNODB STATUS

```sql
-- 查看最近一次死锁信息
SHOW ENGINE INNODB STATUS\G
```

**输出解读（关键部分）：**

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-01-15 10:30:45 0x7f1234
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 10 sec starting index read
mysql tables in use 2, locked 2
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 8, OS thread handle 14000, query id 100 localhost root updating
UPDATE inventory SET stock = stock + 1 WHERE product_id = 100
*** (1) HOLDS THE LOCK(S):
-- 事务B 持有 orders 表上的锁
Record lock, heap no 15 PHYSICAL RECORD: ...

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
-- 事务B 等待 inventory 表上的锁
Record lock, heap no 10 PHYSICAL RECORD: ...

*** (2) TRANSACTION:
TRANSACTION 12344, ACTIVE 15 sec starting index read
mysql tables in use 2, locked 2
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 7, OS thread handle 15000, query id 99 localhost root updating
INSERT INTO orders (id, user_id, product_id, ...) VALUES (...)
*** (2) HOLDS THE LOCK(S):
-- 事务A 持有 inventory 表上的锁
Record lock, heap no 10 PHYSICAL RECORD: ...

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
-- 事务A 等待 orders 表上的锁
Record lock, heap no 15 PHYSICAL RECORD: ...

*** WE ROLL BACK TRANSACTION (2)
-- InnoDB 选择回滚事务A（回滚代价较小的一方）
```

**诊断步骤：**

| 步骤 | 命令 | 目的 |
|------|------|------|
| 1 | `SHOW ENGINE INNODB STATUS\G` | 查看 LATEST DETECTED DEADLOCK 节 |
| 2 | 识别两个事务各自 HOLD 和 WAIT 的锁 | 找出循环等待链 |
| 3 | 分析 SQL 执行顺序 | 确认加锁顺序是否一致 |
| 4 | 检查涉及的索引 | 确认是否因索引不当导致锁范围过大 |

**解决方案：**

```sql
-- 方案1：统一加锁顺序（所有事务先操作 inventory，再操作 orders）
-- 事务A 和 事务B 都按：inventory → orders 顺序

-- 方案2：使用 SELECT ... FOR UPDATE 提前锁定
-- 事务B 提前锁定 inventory，避免循环等待
START TRANSACTION;
SELECT * FROM inventory WHERE product_id = 100 FOR UPDATE;  -- 提前锁
SELECT * FROM orders WHERE product_id = 100 FOR UPDATE;
UPDATE inventory SET stock = stock + 1 WHERE product_id = 100;
COMMIT;

-- 方案3：减小锁粒度，确保 WHERE 条件走索引
-- 避免锁升级为表锁或大范围间隙锁

-- 方案4：设置锁等待超时（防止无限等待）
SET innodb_lock_wait_timeout = 5;  -- 默认 50 秒
```

#### 锁监控

```sql
-- 查看当前锁等待情况
SELECT * FROM performance_schema.data_lock_waits\G

-- 查看当前持有锁的事务
SELECT * FROM performance_schema.data_locks;

-- 查看 InnoDB 状态（含锁信息）
SHOW ENGINE INNODB STATUS\G

-- 查看事务列表
SELECT * FROM information_schema.INNODB_TRX\G
```

### 5.5 锁优化建议

- 尽量走索引，避免行锁升级为表锁
- 控制事务大小，避免长事务
- 统一资源访问顺序（最重要的死锁预防手段）
- 合理使用隔离级别，不盲目使用 SERIALIZABLE
- 监控锁等待，及时优化慢 SQL

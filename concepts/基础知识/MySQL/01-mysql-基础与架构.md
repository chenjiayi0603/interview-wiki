# MySQL 基础与架构

> SQL 分类、MySQL 架构、存储引擎、范式设计、JOIN 类型。

---

## 一、SQL 分类

### 1.1 DDL（数据定义语言）

```sql
CREATE DATABASE db_name;
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(50));
ALTER TABLE users ADD COLUMN age INT;
DROP TABLE users;
```

### 1.2 DML（数据操作语言）

```sql
INSERT INTO users (id, name) VALUES (1, 'Tom');
UPDATE users SET name = 'Jerry' WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

### 1.3 DQL（数据查询语言）

```sql
SELECT id, name FROM users WHERE age > 18 ORDER BY id DESC LIMIT 10;
```

### 1.4 DCL（数据控制语言）

```sql
GRANT SELECT ON db.* TO 'user'@'%';
REVOKE INSERT ON db.* FROM 'user'@'%';
```

### 1.5 TCL（事务控制语言）

```sql
START TRANSACTION;  -- 或 BEGIN
COMMIT;
ROLLBACK;
SAVEPOINT sp1;
```

---

## 二、MySQL 架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────┐
│              客户端/应用                          │
├─────────────────────────────────────────────────┤
│              MySQL Server 层                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ 连接器   │ │ 分析器   │ │ 优化器   │           │
│  │ Connector│ │ Parser  │ │Optimizer│           │
│  └─────────┘ └─────────┘ └────┬────┘           │
│                               │                  │
│  ┌────────────────────────────▼──────────────┐  │
│  │               执行器 Executor               │  │
│  │        调用 Handler API 与引擎交互          │  │
│  └────────────────────────────┬──────────────┘  │
├───────────────────────────────┼─────────────────┤
│          InnoDB 存储引擎层     │                  │
│  ┌────────────────────────────▼──────────────┐  │
│  │              Buffer Pool                    │  │
│  │  数据页 | 索引页 | Undo页 | Change Buffer   │  │
│  └────────────────────────────┬──────────────┘  │
│                               │                  │
│  ┌────────────────────────────▼──────────────┐  │
│  │             磁盘 (.ibd / ibdata1)          │  │
│  └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 2.2 Server 层组件

| 组件 | 功能 |
|------|------|
| **连接器** | TCP 连接、身份认证、权限检查 |
| **分析器** | 词法分析 → Token 分解；语法分析 → AST |
| **优化器** | 选择索引、确定 JOIN 顺序、成本估算 |
| **执行器** | 调用 Handler API 与存储引擎交互 |
| **Binlog** | 逻辑日志，用于主从复制和数据恢复 |

### 2.3 InnoDB 引擎层组件

| 组件 | 功能 |
|------|------|
| **Buffer Pool** | 数据页/索引页缓存，减少磁盘 IO |
| **Redo Log** | 物理日志，保证持久性（WAL） |
| **Undo Log** | 回滚日志，保证原子性和 MVCC |
| **Change Buffer** | 二级索引修改缓存，减少随机读 |
| **Doublewrite Buffer** | 避免数据页部分写入故障 |

---

## 三、存储引擎对比

### 3.1 InnoDB vs MyISAM

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务** | ✅ 支持 | ❌ 不支持 |
| **锁粒度** | 行级锁 | 表级锁 |
| **索引** | 聚簇索引（主键即数据） | 非聚簇索引（索引与数据分离） |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **MVCC** | ✅ 支持 | ❌ 不支持 |
| **COUNT(\*)** | 需扫描索引 | 单独计数，极快 |
| **适用场景** | 高并发写、事务要求 | 读多写少、日志分析 |

```sql
-- 查看存储引擎
SHOW ENGINES;
-- 创建表指定引擎
CREATE TABLE t (id INT) ENGINE = InnoDB;
```

### 3.2 聚簇索引 vs 非聚簇索引

**InnoDB 聚簇索引：**
- 数据行直接存储在索引的叶子节点上
- 通常主键就是聚簇索引
- 每表只能有一个

**InnoDB 二级索引：**
- 叶子节点存储主键值
- 查询时需要回表（先找到主键，再到聚簇索引查数据）

```
二层索引查询路径：
  二级索引 B+Tree 查找 → 获得主键 ID → 回表查聚簇索引 → 完整数据行

覆盖索引（无回表）：
  二级索引 B+Tree 查找 → 索引中已有所有需要的列 → 直接返回
```

**MyISAM 索引：**
- 所有索引都是非聚簇索引
- 索引文件（.MYI）与数据文件（.MYD）分离
- 叶子节点存储数据的物理地址

---

## 四、数据库范式设计

### 4.1 范式概览

| 范式 | 核心要求 | 解决的问题 |
|------|---------|-----------|
| **1NF** | 字段不可再分 | 原子性 |
| **2NF** | 非主属性完全依赖主键 | 部分依赖 |
| **3NF** | 非主属性直接依赖主键 | 传递依赖 |
| **BCNF** | 所有决定因素包含候选键 | 主属性部分依赖 |

### 4.2 示例：2NF 拆分

```sql
-- ❌ 违反 2NF：姓名只依赖于学号（部分依赖）
CREATE TABLE 选课 (
    学号 INT, 课程号 INT, 姓名 VARCHAR(50), 成绩 INT,
    PRIMARY KEY (学号, 课程号)
);

-- ✅ 满足 2NF：拆分为两个表
CREATE TABLE 学生 (学号 INT PRIMARY KEY, 姓名 VARCHAR(50));
CREATE TABLE 选课 (学号 INT, 课程号 INT, 成绩 INT, PRIMARY KEY (学号, 课程号));
```

### 4.3 范式与反范式权衡

**范式化优势：** 减少冗余、避免更新异常、保证一致性
**反范式化场景：** 高频读查询（如报表）、用空间换时间

---

## 五、JOIN 类型详解

### 5.1 INNER JOIN（内连接）

```sql
SELECT * FROM A INNER JOIN B ON A.id = B.a_id;
-- 只返回两表匹配的行
```

### 5.2 LEFT JOIN（左连接）

```sql
SELECT * FROM A LEFT JOIN B ON A.id = B.a_id;
-- 左表全部保留，右表不匹配的列填 NULL
```

### 5.3 RIGHT JOIN（右连接）

```sql
SELECT * FROM A RIGHT JOIN B ON A.id = B.a_id;
-- 右表全部保留，左表不匹配的列填 NULL
-- 等价于：SELECT * FROM B LEFT JOIN A ON B.a_id = A.id;
```

### 5.4 ON vs WHERE 的区别

```sql
-- ON：决定如何连接，左表行始终保留
SELECT * FROM user u
LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 200;
-- 结果：所有用户保留，只连出金额>200的订单

-- WHERE：连接后过滤，会剔除不满足条件的左表行
SELECT * FROM user u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 200;
-- 结果：只保留有订单且金额>200的用户
```

**记忆要点：** ON 决定怎么连，WHERE 决定连完后留哪些。

### 5.5 JOIN 性能优化

- **驱动表要小**：用小表驱动大表
- **被驱动表 join 列要有索引**：Index Nested Loop
- **避免 SELECT \***：只查需要的列，配合覆盖索引
- **EXPLAIN 分析**：关注 type（ref/eq_ref 为佳）

```sql
-- 延迟关联优化大分页
SELECT * FROM order_log t
INNER JOIN (SELECT id FROM order_log ORDER BY id LIMIT 1000000, 20) tmp
  ON t.id = tmp.id;
```

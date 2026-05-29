# MySQL 连接详解

## 左连接与右连接示例与说明

### 概念

- **LEFT JOIN（左连接）**：保留**左表**全部行，右表能匹配则填值，不能匹配则右表列用 `NULL`。
- **RIGHT JOIN（右连接）**：保留**右表**全部行，左表能匹配则填值，不能匹配则左表列用 `NULL`。  
  **RIGHT JOIN 等价于“表顺序调换后的 LEFT JOIN”**，实际常用 LEFT JOIN 统一写法。

### 示例数据

```sql
-- 用户表 user(id, name)
1, '张三'
2, '李四'
3, '王五'

-- 订单表 orders(id, user_id, amount)
101, 1, 100
102, 1, 200
103, 2, 300
104, 99, 400   -- user_id=99 在 user 中不存在
```

### LEFT JOIN 示例

**需求**：所有用户及其订单（没有订单也要显示用户）。

```sql
SELECT u.id AS user_id, u.name, o.id AS order_id, o.amount
FROM user u
LEFT JOIN orders o ON u.id = o.user_id;
```

| user_id | name | order_id | amount |
|---------|------|----------|--------|
| 1 | 张三 | 101 | 100 |
| 1 | 张三 | 102 | 200 |
| 2 | 李四 | 103 | 300 |
| 3 | 王五 | NULL | NULL |

左表 `user` 全部保留；王五无订单，右表列为 NULL。

### RIGHT JOIN 示例

**需求**：所有订单及其用户（即使用户不存在也要显示订单）。

```sql
SELECT u.id AS user_id, u.name, o.id AS order_id, o.amount
FROM user u
RIGHT JOIN orders o ON u.id = o.user_id;
```

| user_id | name | order_id | amount |
|---------|------|----------|--------|
| 1 | 张三 | 101 | 100 |
| 1 | 张三 | 102 | 200 |
| 2 | 李四 | 103 | 300 |
| NULL | NULL | 104 | 400 |

右表 `orders` 全部保留；user_id=99 的订单无对应用户，左表列为 NULL。

等价写法（只用 LEFT JOIN）：

```sql
FROM orders o
LEFT JOIN user u ON u.id = o.user_id;
```

### ON 与 WHERE 的区别（重要）

- **条件写在 ON**：只影响“如何连接”，不满足条件的右表行会变成 NULL，但**左表行仍保留**。
- **条件写在 WHERE**：在连接结果上再过滤，会**过滤掉右表为 NULL 的行**，效果上相当于把 LEFT JOIN 变成 INNER JOIN。

**例：只保留“有金额>200 的订单”的用户（无订单或订单不满足则不要该用户）**

```sql
-- 用 WHERE：左表行也会被过滤掉
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 200;
```

**例：保留所有用户，只把金额>200 的订单连出来（没有则订单列为 NULL）**

```sql
-- 用 ON：左表全部保留
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM user u
LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 200;
```

面试可答：**ON 决定怎么连、是否保留左/右表行；WHERE 在连接后过滤，会丢弃不满足条件的行（包括因左/右连接产生的 NULL 行）。**

---

## 与 PostgreSQL 的简要对比

| 维度 | MySQL | PostgreSQL |
|------|--------|------------|
| 进程/线程 | 一连接一线程 | 一连接一进程 |
| 存储引擎 | Server + 可插拔（如 InnoDB） | 单一内置引擎 |
| MVCC | undo log + 隐藏列 + 版本链 | 表内多版本 tuple + xmin/xmax + autovacuum |
| 日志与复制 | redo + binlog，两阶段提交 | 统一 WAL，恢复与复制共用 |

[src: raw/ingested/2技术/mysql/MySQL 连接详解.md]

## Related Pages
- [[SQL连接与集合操作]]
- [[SQL聚合查询与面试题]]
- [[SQL基础查询]]
- [[SQL子查询与窗口函数]]
- [[SQL数据修改与删除]]
- [[SQL高级DML]]

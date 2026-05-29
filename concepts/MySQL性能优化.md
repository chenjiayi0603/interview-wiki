# MySQL性能优化

See also: [[MySQL主从复制与高可用]]

## 索引优化
- 为WHERE、JOIN、ORDER BY、GROUP BY列建索引
- 使用覆盖索引避免回表
- 遵循最左前缀原则

## SQL优化
- 避免SELECT *、函数计算、LIKE '%xxx'
- 使用JOIN替代子查询
- 合理使用LIMIT

## EXPLAIN关键字段

| type类型 | 说明 | 扫描方式 | 性能 |
|---------|------|---------|------|
| **system** | 系统表，只有一行数据 | 直接访问 | 最优 |
| **const** | 通过主键或唯一索引查询，最多返回一行 | 索引唯一扫描 | 极优 |
| **eq_ref** | 唯一索引扫描，每个索引键值对应表中唯一一行 | 索引唯一扫描 | 极优 |
| **ref** | 非唯一索引扫描，返回匹配某个值的所有行 | 索引范围扫描 | 优秀 |
| **range** | 索引范围扫描，只检索给定范围的行 | 索引范围扫描 | 良好 |
| **index** | 全索引扫描，遍历整个索引树 | 全索引扫描 | 一般 |
| **ALL** | 全表扫描，遍历整个表 | 全表扫描 | 最差 |

**Extra关键值**：
- `Using index`：覆盖索引
- `Using where`：WHERE过滤
- `Using index condition`：索引下推

## 慢查询优化

### 开启慢查询日志
```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- 设置慢查询阈值（秒）
```

### 优化案例

**案例1：索引缺失**
```sql
-- 慢查询
SELECT * FROM orders WHERE user_id = 123 AND status = 'paid';
-- 优化：创建联合索引
CREATE INDEX idx_user_status ON orders(user_id, status);
```

**案例2：全表扫描**
```sql
-- 慢查询
SELECT * FROM users WHERE YEAR(create_time) = 2023;
-- 优化：避免在列上使用函数
SELECT * FROM users WHERE create_time >= '2023-01-01' AND create_time < '2024-01-01';
```

**案例3：子查询优化**
```sql
-- 慢查询
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 1000);
-- 优化：使用JOIN
SELECT u.* FROM users u INNER JOIN orders o ON u.id = o.user_id WHERE o.amount > 1000;
```

## JOIN 性能优化

### JOIN 大表的复杂度
- **最坏情况（无索引、嵌套全表扫描）**：复杂度接近 O(N × M)
- **有索引的常见情况（索引嵌套循环）**：复杂度大致为 O(N_filtered × log M)

### 优化要点
1. **为 JOIN 和过滤条件建立合适索引**
2. **让驱动表尽量"小"**：先用子查询/派生表对大表过滤，再与其他表 JOIN
3. **避免不必要的 SELECT \***：只查询真正需要的列，减少回表次数
4. **使用 EXPLAIN 诊断 JOIN 问题**：关注 type、rows、Extra 等字段

> **记忆要点**：JOIN 大表优化的核心就是两件事——**驱动表要小，被驱动表要有索引**，再用 EXPLAIN 验证执行计划是否符合预期。

[src: raw/ingested/2技术/mysql/mysql复习文档.md]

## Related Pages
- [[MySQL主从复制与高可用]]
- [[MySQL索引]]
- [[MVCC多版本并发控制]]
- [[MySQL存储引擎]]
- [[MySQL分库分表]]
- [[MySQL锁机制]]
- [[性能优化]]
- [[数据库范式]]
- [[MySQL索引优化实用技巧]]

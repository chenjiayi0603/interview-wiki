# MySQL 索引优化

> B+Tree 原理、索引类型、10 大优化技巧、最左前缀、覆盖索引、索引下推、EXPLAIN 分析。

---

## 一、索引底层结构

### 1.1 B+Tree 的特点

```
B+Tree vs B-Tree 核心区别：
┌─────────────────────────────────────────────────────────────┐
│                    B-Tree            B+Tree                  │
├─────────────────────────────────────────────────────────────┤
│  数据存储       所有节点都存数据      只有叶子节点存数据       │
│  叶子节点        无链表              有双向链表（范围查询）    │
│  范围查询        慢（需中序遍历）     快（链表顺序扫）         │
│  高度            高                  低（非叶子节点只存键）   │
└─────────────────────────────────────────────────────────────┘
```

**InnoDB 为何用 B+Tree：**
- 非叶子节点只存键，可容纳更多条目，降低树高（一般 3~4 层）
- 叶子节点双向链表，范围查询极快
- 所有数据都在叶子节点，查询次数稳定

### 1.2 索引类型分类

| 索引类型 | 特性 | 个数限制 |
|---------|------|---------|
| **聚簇索引**（主键索引） | 叶子节点存数据行 | 每表 1 个 |
| **普通索引** | 叶子节点存主键值，需回表 | 多个 |
| **唯一索引** | 值唯一，允许 NULL | 多个 |
| **联合索引** | 多列组合，最左前缀原则 | 多个 |
| **全文索引** | 倒排索引，用于文本搜索 | 多个 |
| **空间索引** | R-Tree，地理空间数据 | 多个 |

---

## 二、索引优化 10 大技巧

### 技巧 1：高选择性原则

选择性 = COUNT(DISTINCT col) / COUNT(\*)，越接近 1 效果越好。

```sql
-- 计算选择性
SELECT COUNT(DISTINCT phone) / COUNT(*) AS selectivity FROM users;
-- > 0.2 适合建索引，< 0.1 效果差
-- 主键/手机号 = 1.0（最佳），性别 ≈ 0.5（不适合单独索引）
```

### 技巧 2：最左前缀原则

联合索引 `(a, b, c)` 相当于创建了 `(a)`、`(a,b)`、`(a,b,c)` 三个索引。

```sql
CREATE INDEX idx_name_age_city ON users(name, age, city);

-- ✅ 有效
WHERE name = 'Tom'
WHERE name = 'Tom' AND age = 25
WHERE name = 'Tom' AND age = 25 AND city = 'Beijing'

-- ❌ 无效（跳过最左列）
WHERE age = 25
WHERE city = 'Beijing'
WHERE age = 25 AND city = 'Beijing'

-- △ 只能用部分索引
WHERE name = 'Tom' AND city = 'Beijing'  -- 只用 name，city 无法用
```

**联合索引列顺序：** 等值查询列放前面 → 范围查询列放后面 → 高选择性优先

### 技巧 3：覆盖索引避免回表

当查询的所有列都在索引中时，直接从索引返回数据，无需回表。

```sql
-- 索引 (user_id, amount)
CREATE INDEX idx_user_amount ON orders(user_id, amount);

-- ✅ 覆盖索引（Extra: Using Index）
EXPLAIN SELECT user_id, amount FROM orders WHERE user_id = 1001;

-- ❌ 需要回表（status 不在索引中）
EXPLAIN SELECT user_id, amount, status FROM orders WHERE user_id = 1001;
```

### 技巧 4：避免索引失效的 8 大陷阱

| 失效场景 | ❌ 错误写法 | ✅ 正确写法 |
|---------|-----------|-----------|
| 函数操作列 | `WHERE YEAR(date)=2024` | `WHERE date>='2024-01-01' AND date<'2025-01-01'` |
| 隐式类型转换 | `WHERE phone=13800000000` | `WHERE phone='13800000000'` |
| LIKE 以通配符开头 | `WHERE name LIKE '%张%'` | `WHERE name LIKE '张%'` |
| OR 连接非索引列 | `WHERE id=1 OR status=1` | 用 UNION 拆分 |
| NOT/!=/<> | `WHERE status!='ok'` | `WHERE status IN ('a','b')` |
| 联合索引跳列 | `WHERE age=25`（索引是 name,age） | 带上最左列 |
| 数据量太小 | 全表扫描比索引快 | 优化器自动选择 |
| 计算操作 | `WHERE price+10>100` | `WHERE price>90` |

### 技巧 5：利用索引排序

当 ORDER BY 列与索引列顺序一致时，避免 filesort。

```sql
CREATE INDEX idx_user_time ON orders(user_id, create_time);

-- ✅ 利用索引排序（Extra 无 filesort）
SELECT * FROM orders WHERE user_id = 1001 ORDER BY create_time;

-- ❌ filesort
SELECT * FROM orders WHERE user_id = 1001 ORDER BY amount;
```

### 技巧 6：前缀索引节省空间

适用于长字符串列（URL、邮箱）。

```sql
-- 确定合适的前缀长度
SELECT
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel_10,
    COUNT(DISTINCT email) / COUNT(*) AS sel_full
FROM users;
-- 选择与满索引选择性接近的前缀长度

-- 创建前缀索引
CREATE INDEX idx_email_prefix ON users(email(10));
```

**注意：** 前缀索引无法用于 ORDER BY 和覆盖索引。

### 技巧 7：EXPLAIN 分析执行计划

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1001 AND status = 'pending';
```

**关键字段：**

| 字段 | 含义 | 优化目标 |
|------|------|---------|
| **type** | 访问类型 | 至少 range，避免 ALL |
| **key** | 实际使用的索引 | 确认走了正确索引 |
| **rows** | 预估扫描行数 | 越小越好 |
| **Extra** | 额外信息 | 关注 Using index / Using filesort |

**type 性能排序：** system > const > eq_ref > ref > range > index > ALL

**Extra 关键标识：**
- `Using Index` ✅ 覆盖索引
- `Using index condition` ✅ 索引下推（ICP）
- `Using filesort` ❌ 需额外排序
- `Using temporary` ❌ 使用临时表

### 技巧 8：索引下推（ICP）

MySQL 5.6+ 优化，将 WHERE 条件下推到存储引擎层过滤。

```sql
-- 索引 (name, age)
SELECT * FROM users WHERE name LIKE '张%' AND age = 25;

-- 无 ICP：先找所有"张%"的行，回表后再过滤 age=25
-- 有 ICP：在索引层直接过滤 age=25，减少回表次数
-- EXPLAIN 显示 Extra: Using index condition
```

### 技巧 9：合理控制索引数量

- 单表建议不超过 **5~6 个**索引
- 联合索引列不超过 **5 列**
- 索引会降低 INSERT/UPDATE/DELETE 性能（10%~40%）

```sql
-- 查找未使用的索引（MySQL 5.6+）
SELECT object_schema, object_name, index_name
FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql', 'sys');
```

### 技巧 10：定期维护索引健康

```sql
-- 查看索引碎片率
SELECT table_name, index_name,
    ROUND(data_free / (data_length + index_length) * 100, 2) AS frag_ratio
FROM information_schema.tables
WHERE table_schema = 'your_db' AND data_free > 0;

-- 重建索引（整理碎片）
ALTER TABLE orders ENGINE=InnoDB;  -- 在线 DDL
OPTIMIZE TABLE orders;             -- 锁表，适合小表

-- 更新统计信息
ANALYZE TABLE orders;
```

---

## 三、索引设计黄金法则

1. **先 EXPLAIN，再加索引** —— 不要盲目加
2. **覆盖索引优先** —— 能避免回表就避免
3. **联合索引设计** —— 等值在前，范围在后
4. **控制数量** —— 单表不超过 6 个索引
5. **定期审计** —— 删除无用索引，维护统计信息

```sql
-- 典型索引设计示例
-- WHERE status = 'active' AND create_time > '2024-01-01'
-- ORDER BY create_time DESC
-- LIMIT 10
CREATE INDEX idx_status_time ON orders(status, create_time);
-- status 等值查询在前，create_time 范围+排序在后
```

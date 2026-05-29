# MySQL索引优化实用技巧

> 参考链接：https://zhuanlan.zhihu.com/p/671404492

以下是 MySQL 索引优化的 10 个核心实用技巧，涵盖从设计到诊断的完整优化体系。

---

## 技巧 1：选择合适的索引列 - 高选择性原则

**核心思想**：索引列的选择性（Selectivity）越高，索引效果越好。

```sql
-- 计算列的选择性
SELECT 
    COUNT(DISTINCT column_name) / COUNT(*) AS selectivity
FROM table_name;

-- 选择性参考标准：
-- > 0.2  ：适合建索引
-- < 0.1  ：索引效果差
-- = 1    ：唯一索引，最佳
```

**实战示例**：

```sql
-- ❌ 差的索引选择（性别列，选择性约0.5）
CREATE INDEX idx_gender ON users(gender);

-- ✅ 好的索引选择（手机号，选择性接近1）
CREATE INDEX idx_phone ON users(phone);

-- ✅ 订单状态+时间联合索引（提高整体选择性）
CREATE INDEX idx_status_time ON orders(status, create_time);
```

**选择性对比表**：

| 列类型 | 典型选择性 | 是否适合索引 |
|--------|-----------|-------------|
| 主键/唯一键 | 1.0 | ✅ 最佳 |
| 手机号/邮箱 | 0.95+ | ✅ 优秀 |
| 用户ID | 0.8+ | ✅ 良好 |
| 城市 | 0.01~0.1 | △ 视情况 |
| 性别/状态 | 0.001~0.01 | ❌ 不推荐单独索引 |

---

## 技巧 2：联合索引的最左前缀原则

**核心规则**：联合索引 `(a, b, c)` 相当于创建了 3 个索引：`(a)`、`(a,b)`、`(a,b,c)`

```
联合索引 idx(a, b, c) 的使用规则：

┌─────────────────────────────────────────────────────────────┐
│  查询条件              │ 是否使用索引 │ 说明                │
├─────────────────────────────────────────────────────────────┤
│  WHERE a = 1           │     ✅      │ 使用索引 a          │
│  WHERE a = 1 AND b = 2 │     ✅      │ 使用索引 a,b        │
│  WHERE a = 1 AND b = 2 │     ✅      │ 使用完整索引        │
│        AND c = 3       │             │                     │
│  WHERE b = 2           │     ❌      │ 缺少最左列 a        │
│  WHERE c = 3           │     ❌      │ 缺少最左列 a        │
│  WHERE b = 2 AND c = 3 │     ❌      │ 缺少最左列 a        │
│  WHERE a = 1 AND c = 3 │     △      │ 只用到 a，c 无法用  │
└─────────────────────────────────────────────────────────────┘
```

**实战应用**：

```sql
-- 创建联合索引
CREATE INDEX idx_name_age_city ON users(name, age, city);

-- ✅ 有效查询
SELECT * FROM users WHERE name = 'Tom';
SELECT * FROM users WHERE name = 'Tom' AND age = 25;
SELECT * FROM users WHERE name = 'Tom' AND age = 25 AND city = 'Beijing';

-- ❌ 无效查询（索引失效）
SELECT * FROM users WHERE age = 25;
SELECT * FROM users WHERE city = 'Beijing';
SELECT * FROM users WHERE age = 25 AND city = 'Beijing';
```

**索引列顺序设计原则**：

```
联合索引列顺序优先级：
1. 等值查询列放前面
2. 范围查询列放后面
3. 排序列考虑放入索引
4. 高选择性列优先

示例：WHERE status = 'active' AND create_time > '2024-01-01'
推荐索引：INDEX(status, create_time)  -- status等值在前
```

---

## 技巧 3：覆盖索引避免回表

**原理**：当查询的所有列都在索引中时，直接从索引返回数据，无需回表查询。

```
普通查询 vs 覆盖索引查询：

普通查询（需要回表）：
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 二级索引    │ ──▶ │ 获取主键    │ ──▶ │ 回表查数据  │
│ B+树查找    │     │ ID值        │     │ 聚簇索引    │
└─────────────┘     └─────────────┘     └─────────────┘
        ↓                 ↓                   ↓
     IO 1次           获得 ID            IO 2次（额外开销）

覆盖索引查询（无需回表）：
┌─────────────┐     ┌─────────────┐
│ 二级索引    │ ──▶ │ 直接返回    │
│ B+树查找    │     │ 索引中数据  │
└─────────────┘     └─────────────┘
        ↓                 ↓
     IO 1次          无需回表（性能最优）
```

**实战示例**：

```sql
-- 表结构
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    product_id INT,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    create_time DATETIME
);

-- 创建覆盖索引
CREATE INDEX idx_user_amount ON orders(user_id, amount);

-- ✅ 覆盖索引查询（Using Index）
EXPLAIN SELECT user_id, amount FROM orders WHERE user_id = 1001;
-- Extra: Using Index  ← 表示使用了覆盖索引

-- ❌ 需要回表的查询
EXPLAIN SELECT user_id, amount, status FROM orders WHERE user_id = 1001;
-- 需要查询 status 列，不在索引中，必须回表
```

**覆盖索引设计技巧**：

```sql
-- 原始查询
SELECT user_id, order_count, total_amount 
FROM user_stats 
WHERE user_id BETWEEN 1000 AND 2000;

-- 优化：创建覆盖索引包含所有查询列
CREATE INDEX idx_cover ON user_stats(user_id, order_count, total_amount);
-- 查询直接从索引获取数据，性能提升 3~5 倍
```

---

## 技巧 4：避免索引失效的常见陷阱

```
索引失效场景全景图：
┌─────────────────────────────────────────────────────────────┐
│                     索引失效的 8 大陷阱                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 函数操作列                                               │
│     WHERE YEAR(create_time) = 2024    ❌                    │
│     WHERE create_time >= '2024-01-01' ✅                    │
│                                                              │
│  2. 隐式类型转换                                             │
│     WHERE phone = 13800138000  ❌ (phone是VARCHAR)          │
│     WHERE phone = '13800138000' ✅                          │
│                                                              │
│  3. 使用 OR 连接非索引列                                     │
│     WHERE indexed_col = 1 OR non_indexed = 2  ❌            │
│     改用 UNION 或分别查询 ✅                                │
│                                                              │
│  4. LIKE 以通配符开头                                        │
│     WHERE name LIKE '%张%'   ❌                             │
│     WHERE name LIKE '张%'    ✅                             │
│                                                              │
│  5. 使用 NOT、!=、<>                                        │
│     WHERE status != 'deleted'  ❌ (可能全表扫描)            │
│     WHERE status IN ('active','pending') ✅                 │
│                                                              │
│  6. IS NULL / IS NOT NULL（视情况）                         │
│     大量NULL值时索引可能失效                                 │
│                                                              │
│  7. 联合索引不满足最左前缀                                   │
│     INDEX(a,b,c) → WHERE b=1  ❌                            │
│                                                              │
│  8. 数据量太小                                               │
│     优化器认为全表扫描更快时会放弃索引                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**优化对照表**：

| 失效写法 | 优化写法 | 说明 |
|---------|---------|------|
| `WHERE YEAR(date)=2024` | `WHERE date>='2024-01-01' AND date<'2025-01-01'` | 避免函数 |
| `WHERE price+10>100` | `WHERE price>90` | 避免计算 |
| `WHERE LOWER(name)='tom'` | `WHERE name='Tom'` 或函数索引 | 避免函数 |
| `WHERE id='123'` (id是INT) | `WHERE id=123` | 类型匹配 |
| `WHERE name LIKE '%tom'` | 使用全文索引或搜索引擎 | 前缀匹配 |

---

## 技巧 5：利用索引进行排序优化

**原理**：当 ORDER BY 的列与索引列顺序一致时，可避免额外排序（filesort）。

```sql
-- 表结构和索引
CREATE INDEX idx_user_time ON orders(user_id, create_time);

-- ✅ 利用索引排序（无 filesort）
SELECT * FROM orders 
WHERE user_id = 1001 
ORDER BY create_time;
-- 索引已按 create_time 有序，直接返回

-- ❌ 无法利用索引排序（产生 filesort）
SELECT * FROM orders 
WHERE user_id = 1001 
ORDER BY amount;
-- amount 不在索引中，需要额外排序

-- ✅ 联合索引优化 ORDER BY + WHERE
CREATE INDEX idx_status_time ON orders(status, create_time);

SELECT * FROM orders 
WHERE status = 'pending' 
ORDER BY create_time DESC 
LIMIT 10;
-- 完美利用索引，无需 filesort
```

**排序索引设计原则**：

```
ORDER BY 索引优化规则：

1. ORDER BY 列顺序与索引列顺序一致
   INDEX(a, b) → ORDER BY a, b  ✅
   INDEX(a, b) → ORDER BY b, a  ❌

2. 排序方向一致
   INDEX(a, b) → ORDER BY a ASC, b ASC   ✅
   INDEX(a, b) → ORDER BY a ASC, b DESC  ❌ (MySQL 8.0前)

3. WHERE 等值 + ORDER BY 可组合
   INDEX(a, b) + WHERE a=1 ORDER BY b  ✅

4. MySQL 8.0+ 支持降序索引
   CREATE INDEX idx_desc ON orders(create_time DESC);
```

---

## 技巧 6：前缀索引节省空间

**适用场景**：长字符串列（如URL、邮箱、地址）

```sql
-- 完整字符串索引（占用空间大）
CREATE INDEX idx_email ON users(email);  -- 假设平均30字节

-- ✅ 前缀索引（节省空间）
CREATE INDEX idx_email_prefix ON users(email(10));  -- 只索引前10字节

-- 如何确定前缀长度？计算不同前缀的选择性
SELECT 
    COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) AS sel_5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel_10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) AS sel_15,
    COUNT(DISTINCT email) / COUNT(*) AS sel_full
FROM users;

-- 结果示例：
-- sel_5: 0.75, sel_10: 0.95, sel_15: 0.99, sel_full: 1.0
-- 选择 sel_10 = 0.95 已足够好，使用前缀长度 10
```

**前缀索引的权衡**：

```
┌─────────────────────────────────────────────────────────────┐
│              前缀索引 优势 vs 劣势                           │
├─────────────────────────────────────────────────────────────┤
│  ✅ 优势：                                                   │
│     • 显著减少索引存储空间                                   │
│     • 提高索引维护效率                                       │
│     • 更多索引页可缓存在内存                                 │
│                                                              │
│  ❌ 劣势：                                                   │
│     • 无法用于 ORDER BY                                      │
│     • 无法用于覆盖索引                                       │
│     • 选择性可能降低                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 技巧 7：使用 EXPLAIN 分析执行计划

**EXPLAIN 关键字段解读**：

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1001 AND status = 'pending';
```

```
EXPLAIN 输出关键字段：
┌─────────────────────────────────────────────────────────────┐
│  字段          │ 含义                │ 优化目标             │
├─────────────────────────────────────────────────────────────┤
│  type          │ 访问类型            │ 至少达到 range       │
│                │ system > const >    │ 避免 ALL            │
│                │ eq_ref > ref >      │                      │
│                │ range > index > ALL │                      │
│                │                     │                      │
│  possible_keys │ 可能使用的索引      │ 确认索引被识别       │
│                │                     │                      │
│  key           │ 实际使用的索引      │ 确认正确索引被使用   │
│                │                     │                      │
│  key_len       │ 索引使用长度        │ 联合索引使用了几列   │
│                │                     │                      │
│  rows          │ 预估扫描行数        │ 越小越好             │
│                │                     │                      │
│  filtered      │ 过滤比例            │ 越大越好(100%最佳)   │
│                │                     │                      │
│  Extra         │ 额外信息            │ 关注关键标识         │
│                │ Using Index         │ ✅ 覆盖索引          │
│                │ Using where         │ 需要回表过滤         │
│                │ Using filesort      │ ❌ 需要额外排序      │
│                │ Using temporary     │ ❌ 使用临时表        │
│                │ Using index condition│ ✅ 索引下推(ICP)    │
└─────────────────────────────────────────────────────────────┘
```

**type 访问类型性能排序**：

```
性能从好到差：
system > const > eq_ref > ref > range > index > ALL

┌─────────┬────────────────────────────────────────────────┐
│ system  │ 表只有一行数据                                 │
│ const   │ 主键或唯一索引等值查询，最多一行               │
│ eq_ref  │ 主键或唯一索引的关联查询，每组合只有一行       │
│ ref     │ 非唯一索引等值查询                             │
│ range   │ 索引范围扫描 (>, <, BETWEEN, IN)              │
│ index   │ 全索引扫描（比ALL好，但仍需优化）              │
│ ALL     │ 全表扫描（最差，必须优化！）                   │
└─────────┴────────────────────────────────────────────────┘
```

---

## 技巧 8：索引下推（Index Condition Pushdown）

**原理**：MySQL 5.6+ 的优化特性，将 WHERE 条件下推到存储引擎层过滤。

```
无 ICP（传统方式）：
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ 存储引擎    │ ──▶│ 返回所有    │ ──▶│ Server层   │
│ 根据索引    │    │ 匹配行      │    │ 再次过滤   │
│ 查找数据    │    │ 到Server层  │    │ WHERE条件  │
└─────────────┘    └─────────────┘    └─────────────┘
      大量数据传输，效率低

有 ICP（索引下推）：
┌─────────────┐    ┌─────────────┐
│ 存储引擎    │ ──▶│ 只返回      │
│ 索引查找 +  │    │ 符合条件的  │
│ WHERE过滤   │    │ 少量数据    │
└─────────────┘    └─────────────┘
      减少数据传输，效率高
```

**示例**：

```sql
-- 索引
CREATE INDEX idx_name_age ON users(name, age);

-- 查询
SELECT * FROM users WHERE name LIKE '张%' AND age = 25;

-- 无ICP：先找所有"张%"的行，回表后再过滤 age=25
-- 有ICP：在索引层直接过滤 age=25，减少回表次数

-- EXPLAIN 显示
Extra: Using index condition  -- 表示启用了索引下推
```

---

## 技巧 9：合理控制索引数量

**索引的代价**：

```
索引的双刃剑效应：
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  ✅ 索引的好处：                                             │
│     • 加速 SELECT 查询                                       │
│     • 加速 ORDER BY 排序                                     │
│     • 加速 JOIN 关联                                         │
│                                                              │
│  ❌ 索引的代价：                                             │
│     • INSERT 性能下降 10%~30%                                │
│     • UPDATE 性能下降 10%~40%（需更新索引）                  │
│     • DELETE 性能下降 10%~30%                                │
│     • 占用额外磁盘空间（索引大小约为数据 10%~30%）           │
│     • 增加优化器选择复杂度                                   │
│                                                              │
│  📊 建议指标：                                               │
│     • 单表索引数量：不超过 5~6 个                            │
│     • 联合索引列数：不超过 5 列                              │
│     • 定期清理未使用的索引                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**查找未使用的索引**：

```sql
-- MySQL 5.6+ 通过 sys 库查找未使用索引
SELECT 
    object_schema,
    object_name,
    index_name
FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql', 'sys');

-- 或通过 performance_schema
SELECT 
    object_schema,
    object_name, 
    index_name,
    count_read,
    count_write
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_read = 0
ORDER BY object_schema, object_name;
```

---

## 技巧 10：定期维护索引健康

**索引碎片化问题**：

```
索引碎片化影响：
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  碎片化原因：                                                │
│  • 大量 DELETE 操作留下空洞                                  │
│  • UPDATE 导致行迁移                                         │
│  • 页分裂产生不连续存储                                      │
│                                                              │
│  碎片化后果：                                                │
│  • 索引占用空间增大                                          │
│  • 查询需要更多 I/O                                          │
│  • 缓冲池利用率下降                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**索引维护命令**：

```sql
-- 1. 查看索引碎片率
SELECT 
    table_name,
    index_name,
    ROUND(data_free / (data_length + index_length) * 100, 2) AS frag_ratio
FROM information_schema.tables
WHERE table_schema = 'your_database'
  AND data_free > 0;

-- 2. 重建索引（整理碎片）
-- 方式1：OPTIMIZE TABLE（锁表，适合小表）
OPTIMIZE TABLE orders;

-- 方式2：ALTER TABLE（在线DDL，MySQL 5.6+）
ALTER TABLE orders ENGINE=InnoDB;

-- 方式3：重建单个索引
ALTER TABLE orders DROP INDEX idx_user_id, ADD INDEX idx_user_id(user_id);

-- 3. 更新统计信息
ANALYZE TABLE orders;
```

**索引维护清单**：

```
定期索引维护清单：
┌─────────────────────────────────────────────────────────────┐
│  周期      │ 维护项目                 │ 命令/方法            │
├─────────────────────────────────────────────────────────────┤
│  每日      │ 监控慢查询日志           │ slow_query_log       │
│  每周      │ 检查索引使用情况         │ sys.schema_unused_*  │
│  每月      │ 分析表统计信息           │ ANALYZE TABLE        │
│  每季度    │ 评估索引碎片化           │ OPTIMIZE TABLE       │
│  半年      │ 全面索引审计             │ EXPLAIN + 压测       │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 总结：索引优化十大技巧速查表

| 序号 | 技巧 | 核心要点 | 性能提升 |
|------|------|---------|---------|
| 1 | 高选择性原则 | 选择性 > 0.2 的列建索引 | ★★★★★ |
| 2 | 最左前缀原则 | 联合索引顺序设计 | ★★★★★ |
| 3 | 覆盖索引 | 避免回表，Using Index | ★★★★☆ |
| 4 | 避免索引失效 | 函数、类型转换、LIKE%前缀 | ★★★★★ |
| 5 | 排序优化 | ORDER BY 列与索引一致 | ★★★☆☆ |
| 6 | 前缀索引 | 长字符串用前缀索引 | ★★★☆☆ |
| 7 | EXPLAIN分析 | 关注 type、key、Extra | ★★★★★ |
| 8 | 索引下推 | Using index condition | ★★★☆☆ |
| 9 | 控制索引数量 | 单表不超过5-6个索引 | ★★★★☆ |
| 10 | 定期维护 | ANALYZE + OPTIMIZE | ★★★☆☆ |

---

**💡 黄金法则**：

1. **先 EXPLAIN，再优化** —— 不要盲目加索引
2. **覆盖索引优先** —— 能避免回表就避免
3. **联合索引设计** —— 等值在前，范围在后
4. **定期审计** —— 删除无用索引，维护索引健康
5. **测试验证** —— 任何优化都要在测试环境验证效果

[src: raw/ingested/2技术/mysql/MySQL索引优化实用技巧.md]

## Related Pages
- [[MySQL索引]]
- [[MySQL SQL优化]]
- [[MySQL性能优化]]
- [[MySQL架构与存储引擎]]

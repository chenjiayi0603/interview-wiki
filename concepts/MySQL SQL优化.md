# MySQL SQL 优化

## 一、EXPLAIN 分析

```sql
EXPLAIN SELECT * FROM t WHERE ...
-- id: 执行顺序
-- type: const/eq_ref/ref/range/index/all
-- key: 实际使用的索引
-- rows: 预计扫描行数
-- Extra: Using index/Using filesort/Using temporary
```

## 二、慢查询优化

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- 分析慢查询
EXPLAIN FORMAT=JSON 'SELECT ...'

-- 使用索引
CREATE INDEX idx_name ON t(column)

-- 分页优化
-- 低效：SELECT * FROM t LIMIT 1000000, 10
-- 高效：SELECT * FROM t WHERE id > 1000000 LIMIT 10
```

## 三、SQL 编写规范

- 避免SELECT *
- 批量INSERT
- 合理使用LIMIT
- 预编译语句防注入
- 分析后再添加索引

## 四、面试高频问题

### Q1: 如何优化深度分页？

```sql
-- 低效：OFFSET过大
SELECT * FROM t ORDER BY id LIMIT 1000000, 10

-- 优化1：基于ID游标
SELECT * FROM t WHERE id > 1000000 ORDER BY id LIMIT 10

-- 优化2：延迟关联
SELECT t.* FROM t 
INNER JOIN (SELECT id FROM t ORDER BY id LIMIT 1000000, 10) AS t2 
ON t.id = t2.id

-- 优化3：记录上次查询最大ID
```

[src: raw/ingested/2技术/mysql/MySQL知识.md]

## Related Pages
- [[MySQL索引]]
- [[SQL基础查询]]
- [[SQL子查询与窗口函数]]
- [[SQL连接与集合操作]]
- [[SQL数据修改与删除]]
- [[SQL高级DML]]
- [[SQL聚合查询与面试题]]

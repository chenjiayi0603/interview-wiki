# SQL 数据修改与删除

## UPDATE + CASE

### 627. 变更性别

**表：** `Salary`（id, name, sex, salary），sex 为 'm'/'f'。

**题意：** 一条 UPDATE 交换所有 'm' 与 'f'，且不用 SELECT、不建临时表。

```sql
UPDATE Salary
SET sex = CASE sex WHEN 'm' THEN 'f' ELSE 'm' END;

-- 其他写法
UPDATE Salary SET sex = IF(sex = 'm', 'f', 'm');
UPDATE Salary SET sex = CHAR(ASCII('m') + ASCII('f') - ASCII(sex));
```

## DELETE + 子查询

### 196. 删除重复的电子邮箱

**表：** `Person`（id, email）。

**题意：** 删除重复邮箱，只保留每个邮箱 id 最小的那一行。

```sql
DELETE FROM Person
WHERE id NOT IN (
  SELECT id FROM (
    -- 取每个邮箱的最小 id（即只保留每个邮箱的最早那行）
    SELECT MIN(id) AS id
    FROM Person
    GROUP BY email
  ) t
);
```

注意：MySQL 中 DELETE 时不能直接引用同一张表的子查询，需再包一层子查询取 id。

[src: raw/ingested/2技术/mysql/leetcode-sql.md]

## Related Pages
- [[SQL基础查询]]
- [[SQL子查询与窗口函数]]
- [[SQL连接与集合操作]]

# SQL 子查询与窗口函数

## SELECT + LIMIT / 子查询

### 176. 第二高的薪水

**表：** `Employee`（id, salary）。

**题意：** 第二高薪水，不存在则返回 null。

```sql
-- 子查询包一层才能返回 null
SELECT (
  SELECT DISTINCT salary
  FROM Employee
  ORDER BY salary DESC
  LIMIT 1 OFFSET 1
) AS SecondHighestSalary;
```

### 177. 第 N 高的薪水

**表：** `Employee`（id, salary）。要求写函数 `getNthHighestSalary(N)`，无第 N 高则返回 null。

```sql
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  SET N = N - 1;
  RETURN (
    SELECT DISTINCT salary
    FROM Employee
    ORDER BY salary DESC
    LIMIT 1 OFFSET N
  );
END
```

## 窗口函数 / 排名

### 178. 分数排名

**表：** `Scores`（id, score）。

**题意：** 按分数降序排名，同分同排名，且排名连续无空缺（dense_rank）。

```sql
-- 写法1：窗口函数
SELECT score,
       DENSE_RANK() OVER (ORDER BY score DESC) AS 'Rank'
FROM Scores;

-- 写法2：子查询计数不同分数
SELECT s.score,
       (SELECT COUNT(DISTINCT score) FROM Scores WHERE score >= s.score) AS 'Rank'
FROM Scores s
ORDER BY s.score DESC;
```

## 子查询比较

### 181. 超过经理收入的员工

**表：** `Employee`（id, name, salary, managerId）。

**题意：** 工资高于其经理的员工姓名。

```sql
-- 查找工资高于自己经理的员工姓名
SELECT e.name AS Employee
FROM Employee e
WHERE e.salary > (
  SELECT salary FROM Employee a WHERE a.id = e.managerId
);
-- 子查询查找该员工的经理的工资作为比较对象
```

-- 复杂度分析：
-- 假设 n 为员工总数。对于每个员工，子查询会查找其经理的工资（Employee表通过主键a.id = e.managerId）。如果managerId有索引，子查询为O(1)；否则O(n)。
-- 总体复杂度接近 O(n) ~ O(n log n)（取决于索引）。若无索引，最坏O(n^2)。
-- 实际大多数面试或 LeetCode 环境，主键有索引，复杂度为 O(n)。

[src: raw/ingested/2技术/mysql/leetcode-sql.md]

## Related Pages
- [[SQL基础查询]]
- [[SQL连接与集合操作]]
- [[SQL数据修改与删除]]

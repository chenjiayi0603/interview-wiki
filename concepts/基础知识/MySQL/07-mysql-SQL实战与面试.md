# MySQL SQL 实战与面试

> LeetCode SQL 题解、聚合查询、多表查询、高频面试 Q&A。

---

## 一、LeetCode SQL 题解

### 1.1 WHERE 基础

**595. 大的国家** [🔗](https://leetcode.cn/problems/big-countries/)

```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000 OR population >= 25000000;
```

**1757. 可回收且低脂的产品** [🔗](https://leetcode.cn/problems/recyclable-and-low-fat-products/)

```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y';
```

**584. 寻找用户推荐人** [🔗](https://leetcode.cn/problems/find-customer-referee/)

```sql
-- 注意 NULL 处理
SELECT name FROM customer
WHERE referee_id <> 2 OR referee_id IS NULL;
```

### 1.2 SELECT + LIMIT / 子查询

**176. 第二高的薪水** [🔗](https://leetcode.cn/problems/second-highest-salary/)

```sql
-- 子查询包一层才能返回 null
SELECT (
    SELECT DISTINCT salary
    FROM Employee
    ORDER BY salary DESC
    LIMIT 1 OFFSET 1
) AS SecondHighestSalary;
```

**177. 第 N 高的薪水** [🔗](https://leetcode.cn/problems/nth-highest-salary/)

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
END;
```

### 1.3 LEFT JOIN

**175. 组合两个表** [🔗](https://leetcode.cn/problems/combine-two-tables/)

```sql
SELECT p.FirstName, p.LastName, a.City, a.State
FROM Person p
LEFT JOIN Address a ON p.PersonId = a.PersonId;
```

**183. 从不订购的客户** [🔗](https://leetcode.cn/problems/customers-who-never-order/)

```sql
SELECT c.Name AS Customers
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.Id
WHERE o.Id IS NULL;

-- 等价写法
SELECT c.Name AS Customers
FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.Id);
```

### 1.4 GROUP BY + HAVING

**596. 超过 5 名学生的课** [🔗](https://leetcode.cn/problems/classes-more-than-5-students/)

```sql
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(DISTINCT student) >= 5;  -- 注意去重
```

**182. 查找重复的电子邮箱** [🔗](https://leetcode.cn/problems/duplicate-emails/)

```sql
SELECT email FROM Person
GROUP BY email
HAVING COUNT(email) > 1;
```

### 1.5 自连接 + 日期/连续

**197. 上升的温度** [🔗](https://leetcode.cn/problems/rising-temperature/)

```sql
SELECT a.Id
FROM Weather a
JOIN Weather b ON DATEDIFF(a.recordDate, b.recordDate) = 1
    AND a.Temperature > b.Temperature;
```

**180. 连续出现的数字** [🔗](https://leetcode.cn/problems/consecutive-numbers/)

```sql
SELECT DISTINCT a.Num AS ConsecutiveNums
FROM Logs a, Logs b, Logs c
WHERE a.Id = b.Id - 1
  AND b.Id = c.Id - 1
  AND a.Num = b.Num
  AND b.Num = c.Num;
```

### 1.6 JOIN + 子查询

**184. 部门工资最高的员工** [🔗](https://leetcode.cn/problems/department-highest-salary/)

```sql
SELECT d.name AS Department, e.name AS Employee, e.salary
FROM Employee e
JOIN Department d ON e.departmentId = d.id
WHERE (e.departmentId, e.salary) IN (
    SELECT departmentId, MAX(salary)
    FROM Employee
    GROUP BY departmentId
);
```

### 1.7 窗口函数

**178. 分数排名** [🔗](https://leetcode.cn/problems/rank-scores/)

```sql
SELECT score,
    DENSE_RANK() OVER (ORDER BY score DESC) AS 'Rank'
FROM Scores;
```

### 1.8 UPDATE + CASE

**627. 变更性别** [🔗](https://leetcode.cn/problems/swap-salary/)

```sql
UPDATE Salary
SET sex = CASE sex WHEN 'm' THEN 'f' ELSE 'm' END;
```

### 1.9 DELETE + 子查询

**196. 删除重复的电子邮箱** [🔗](https://leetcode.cn/problems/delete-duplicate-emails/)

```sql
DELETE FROM Person
WHERE id NOT IN (
    SELECT id FROM (
        SELECT MIN(id) AS id
        FROM Person
        GROUP BY email
    ) t  -- 多包一层子查询，避免 MySQL 限制
);
```

---

## 二、聚合查询实战

### 2.1 基础聚合

```sql
-- 查询平均成绩大于 60 分学生的学号和平均成绩
SELECT 学号, AVG(成绩) AS 平均成绩
FROM score
GROUP BY 学号
HAVING AVG(成绩) > 60;

-- 查询至少选修两门课程的学生学号
SELECT 学号, COUNT(课程号) AS 选修数
FROM score
GROUP BY 学号
HAVING COUNT(课程号) >= 2;

-- 查询同名同姓学生名单并统计人数
SELECT 姓名, COUNT(*) AS 人数
FROM student
GROUP BY 姓名
HAVING COUNT(*) >= 2;
```

### 2.2 条件聚合（CASE WHEN）

```sql
-- 查询每门课程的及格人数和不及格人数
SELECT 课程号,
    SUM(CASE WHEN 成绩 >= 60 THEN 1 ELSE 0 END) AS 及格人数,
    SUM(CASE WHEN 成绩 < 60 THEN 1 ELSE 0 END) AS 不及格人数
FROM score
GROUP BY 课程号;
```

### 2.3 排序 + 聚合

```sql
-- 查询每门课程的平均成绩，结果按平均成绩升序，相同时按课程号降序
SELECT 课程号, AVG(成绩) AS 平均成绩
FROM score
GROUP BY 课程号
ORDER BY 平均成绩 ASC, 课程号 DESC;

-- 统计每门课程的选修人数（超过 2 人才统计）
SELECT 课程号, COUNT(学号) AS 选修人数
FROM score
GROUP BY 课程号
HAVING COUNT(学号) > 2
ORDER BY 选修人数 DESC, 课程号 ASC;
```

---

## 三、多表查询实战

### 3.1 多表 LEFT JOIN + 聚合

```sql
-- 查询所有学生的学号、姓名、选课数、总成绩
SELECT a.学号, a.姓名,
    COUNT(b.课程号) AS 选课数,
    SUM(b.成绩) AS 总成绩
FROM student a
LEFT JOIN score b ON a.学号 = b.学号
GROUP BY a.学号;

-- 查询平均成绩大于 85 的学生信息
SELECT a.学号, a.姓名, AVG(b.成绩) AS 平均成绩
FROM student a
LEFT JOIN score b ON a.学号 = b.学号
GROUP BY a.学号
HAVING AVG(b.成绩) > 85;
```

### 3.2 三表连接

```sql
-- 查询学生的选课情况：学号、姓名、课程号、课程名称
SELECT a.学号, a.姓名, c.课程号, c.课程名称
FROM student a
INNER JOIN score b ON a.学号 = b.学号
INNER JOIN course c ON b.课程号 = c.课程号;

-- 查询不同老师所教不同课程平均分从高到低
SELECT a.教师号, a.教师姓名, AVG(c.成绩) AS 平均分
FROM teacher a
INNER JOIN course b ON a.教师号 = b.教师号
INNER JOIN score c ON b.课程号 = c.课程号
GROUP BY a.教师姓名
ORDER BY AVG(c.成绩) DESC;
```

### 3.3 子查询

```sql
-- 查询没有学全所有课的学生的学号、姓名
SELECT 学号, 姓名
FROM student
WHERE 学号 IN (
    SELECT 学号
    FROM score
    GROUP BY 学号
    HAVING COUNT(课程号) < (SELECT COUNT(课程号) FROM course)
);

-- 查询出只选修了两门课程的全部学生的学号和姓名
SELECT 学号, 姓名
FROM student
WHERE 学号 IN (
    SELECT 学号
    FROM score
    GROUP BY 学号
    HAVING COUNT(课程号) = 2
);
```

---

## 四、高频面试 Q&A

### Q1: 描述一条 UPDATE 语句在 MySQL 中的完整执行过程

```
以 UPDATE users SET age = 26 WHERE id = 100 为例：

1. 连接器：接收连接，验证身份权限
2. 分析器：词法语法分析，生成 AST
3. 优化器：选择主键索引（const 级别）
4. 执行器：调用 InnoDB Handler API

【进入 InnoDB 引擎层】
5. 写 Undo Log：记录 age=25（修改前的值）
6. 修改 Buffer Pool：age 25→26，标记脏页
7. 写 Redo Log Buffer：记录页修改

【两阶段提交】
8. Redo Prepare：刷盘到 ib_logfile，标记 PREPARED
9. 写 Binlog：刷盘到 mysql-bin.xxx
10. Redo Commit：写入 Commit 标记

【返回+后台】
11. 返回客户端 "1 row affected"
12. 后台 Page Cleaner 异步刷脏页到 .ibd
```

### Q2: 聚簇索引和二级索引的区别？

| 维度 | 聚簇索引 | 二级索引 |
|------|---------|---------|
| 数据存储 | 叶子节点存完整数据行 | 叶子节点存主键值 |
| 个数 | 每表 1 个 | 可多个 |
| 回表 | 不需要 | 需要（回表查聚簇索引） |
| 确定规则 | 主键→第一个非空唯一索引→隐藏 row_id | 手动创建 |

### Q3: MVCC 如何实现 RC 和 RR？

- **RC**：每次 SELECT 生成新 ReadView → 能看到最新已提交数据
- **RR**：事务开始时生成 ReadView，整个事务复用 → 多次读结果一致

### Q4: Next-Key Lock 如何解决幻读？

Next-Key Lock = Record Lock + Gap Lock。在 RR 级别，SELECT...FOR UPDATE 时，不仅锁定命中的记录，还锁定记录前的间隙，防止其他事务在间隙中插入新行。

### Q5: 死锁怎么处理？

InnoDB 自动检测死锁，回滚其中一个事务（通常选择回滚成本较低的那个）。可通过 `SHOW ENGINE INNODB STATUS` 查看死锁信息。预防方法：按相同顺序访问资源、减少事务大小、合理使用索引。

### Q6: MySQL 主从延迟的原因和解决方案？

**原因：**
- 从库单线程重放（SQL Thread 性能不足）
- 主库大事务（binlog 量大）
- 网络延迟
- 从库执行慢查询

**解决方案：**
- 开启并行复制（slave_parallel_workers）
- 使用半同步复制
- 避免大事务（分批处理）
- 提升从库硬件配置

### Q7: COUNT(\*) 为什么 InnoDB 比 MyISAM 慢？

MyISAM 单独存储了表的总行数（元数据），COUNT(\*) 直接返回。InnoDB 需要扫描索引统计，因为支持 MVCC 不同事务看到不同行数。带上 WHERE 条件时两者都需要扫描。

### Q8: 分库分表后全局 ID 怎么生成？

1. **雪花算法（Snowflake）**：64 位自增 ID，含时间戳+机器ID+序列号
2. **UUID**：全局唯一，但无序，不适合作为聚簇索引
3. **数据库自增**：分段自增（设置不同步长）
4. **Redis incr**：利用 Redis 原子自增

### Q9: 三个数据库范式在面试中怎么答？

- **1NF**：字段不可再分（地址拆分为省/市/区）
- **2NF**：非主属性完全依赖主键（选课表拆学分学生表和成绩表）
- **3NF**：非主属性直接依赖主键，不能传递依赖（学生表的系主任应放到院系表）

### Q10: SQL 优化的一般步骤？

1. 开启慢查询日志，定位慢 SQL
2. EXPLAIN 分析执行计划（关注 type、rows、Extra）
3. 检查索引使用情况（是否失效、是否需要加索引）
4. 改写 SQL（避免函数、隐式转换、SELECT \*）
5. 测试验证效果

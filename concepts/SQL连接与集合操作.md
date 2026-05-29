# SQL 连接与集合操作

## LEFT JOIN

### 175. 组合两个表

**表：** `Person`（personId, firstName, lastName）、`Address`（addressId, personId, city, state）。

**题意：** 每人姓名 + 地址（无地址则为 null）。

```sql
SELECT p.FirstName, p.LastName, a.City, a.State
FROM Person AS p
LEFT JOIN Address AS a ON p.PersonId = a.PersonId;
```

### 183. 从不订购的客户

**表：** `Customers`（Id, Name）、`Orders`（Id, CustomerId）。

**题意：** 从没下过单的客户姓名。

```sql
-- 写法1：LEFT JOIN + NULL
SELECT c.Name AS Customers
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.Id
WHERE o.Id IS NULL;

-- 写法2：NOT EXISTS
SELECT c.Name AS Customers
FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.Id);

-- 写法3：NOT IN
SELECT c.Name AS Customers
FROM Customers c
WHERE c.Id NOT IN (SELECT DISTINCT CustomerId FROM Orders);
```

## NOT IN / 子查询

### 607. 销售员

**表：** `SalesPerson`、`Company`、`Orders`（订单含 com_id、sales_id）。

**题意：** 没有向名为 "RED" 的公司卖过货的销售员姓名。

```sql
SELECT s.name
FROM SalesPerson s
WHERE s.sales_id NOT IN (
  SELECT o.sales_id
  FROM Orders o
  WHERE o.com_id IN (
    SELECT c.com_id FROM Company c WHERE c.name = 'RED'
  )
);
```

## GROUP BY + HAVING

### 596. 超过 5 名学生的课

**表：** `Courses`（student, class），主键 (student, class)。

**题意：** 至少 5 个学生的班级（若同一学生可重复选课，需按题意用 DISTINCT 学生数）。

```sql
SELECT class
FROM Courses
GROUP BY class
-- 为啥 DISTINCT：有可能一个学生会多次选同一门课，不加 DISTINCT 则会重复计数。题意要求“至少 5 个不同学生”，因此要去重。
HAVING COUNT(DISTINCT student) >= 5

-- 或先去重再计数
SELECT class
FROM (SELECT DISTINCT student, class FROM Courses) t
GROUP BY class
HAVING COUNT(*) >= 5;
```

### 182. 查找重复的电子邮箱

**表：** `Person`（Id, Email）。

**题意：** 出现超过 1 次的邮箱。

```sql
-- 写法1
SELECT email FROM Person GROUP BY email HAVING COUNT(email) > 1;

-- 写法2：子查询
SELECT email FROM (
  SELECT email, COUNT(1) AS c FROM Person GROUP BY email
) t WHERE t.c > 1;

-- 写法3：自连接（数据量大时慎用，因为会把表和自身做笛卡尔积后再筛选，行数急剧增加，效率低）
SELECT DISTINCT p1.Email
FROM Person p1
JOIN Person p2 ON p1.Email = p2.Email AND p1.Id != p2.Id;
```

## 自连接 + 日期 / 连续

自连接（Self Join）是指在一张表上进行连接操作，把同一张表作为连接的左右两侧，有别名做区分。常用于需要对表中行之间进行比较、查找连续/相关记录等场景。本质就是一张表和自身的连接（可 inner join 或 left join），语法与普通 join 类似，只是表名相同、别名不同。

常见用法举例：
- 日期序列比对（如找出比昨天温度更高的记录）
- 连续事件检测（如连续出现相同数值）
- 树形结构递归等

例子：比前一天温度高的日期
```sql
SELECT a.Id
FROM Weather a
JOIN Weather b
  ON DATEDIFF(a.recordDate, b.recordDate) = 1
 AND a.Temperature > b.Temperature;
```

这里 a、b 都是 Weather 表，通过设置别名实现自连接。
为什么要这样做？因为 a 和 b 都引用同一张 Weather 表，通过给它们设置不同的别名，可以在一条 SQL 语句中让同张表的两行数据进行比对，从而实现自连接。这常用于需要比较前后日期、连续数据等场景。

### 197. 上升的温度

**表：** `Weather`（id, recordDate, temperature）。

**题意：** 比前一天温度更高的日期 id。

```sql
SELECT a.Id
FROM Weather a
JOIN Weather b ON DATEDIFF(a.recordDate, b.recordDate) = 1 AND a.Temperature > b.Temperature;

-- 或 WHERE 写法
SELECT a.Id
FROM Weather a, Weather b
WHERE DATEDIFF(a.recordDate, b.recordDate) = 1 AND a.Temperature > b.Temperature;
```

### 180. 连续出现的数字

**表：** `Logs`（id, num），id 连续。

**题意：** 至少连续出现 3 次的数字。

```sql
SELECT DISTINCT a.Num AS ConsecutiveNums
FROM Logs a, Logs b, Logs c
WHERE a.Id = b.Id - 1
  AND b.Id = c.Id - 1
  AND a.Num = b.Num
  AND b.Num = c.Num;
```

## JOIN + IN 子查询

### 184. 部门工资最高的员工

**表：** `Employee`（id, name, salary, departmentId）、`Department`（id, name）。

**题意：** 每个部门工资最高的员工（同部门最高薪可能多人）。

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

### 185. 部门工资前三高的所有员工

**表：** 同上。

**题意：** 每个部门中，工资排名前三（按不同工资值算排名）的员工。

```sql
-- 每个部门工资排名前三的所有员工（不同工资算排名, 可能多于3人）
SELECT d.name AS Department, e.name AS Employee, e.salary
-- 连接 Employee 和 Department 表，通过部门 id 实现关联，便于查询每个员工所在部门
FROM Employee e
JOIN Department d ON e.departmentId = d.id
WHERE e.salary IN (
  -- 查询当前部门（e.departmentId）工资排名前三的所有不同工资
  SELECT salary
  FROM (
    -- 查询当前部门的前三高不同工资
    -- 这里并不是自连接，而是相关子查询。下面的 SQL 子句是
    -- “相关子查询”，其中 e.departmentId 来源于外层 Employee e，
    -- 这里通过 WHERE 条件将当前 e 所在部门的工资取出来，按薪水倒序选取该部门前3高的不同工资。
    SELECT DISTINCT salary
    FROM Employee c
    WHERE c.departmentId = e.departmentId
    ORDER BY salary DESC
    LIMIT 3
    -- 选出该部门工资排名前3的不同工资
  ) t
);
```

说明：按“不同工资”排名时需 `DISTINCT salary`；只用 `salary IN` 即可，不必 `(salary, departmentId) IN`。

## JOIN 查询详解

> 参考：https://www.cnblogs.com/pcjim/articles/799302.html

- **left join（左联接）** 返回包括左表中的所有记录和右表中联结字段相等的记录
- **right join（右联接）** 返回包括右表中的所有记录和左表中联结字段相等的记录
- **inner join（等值连接）** 只返回两个表中联结字段相等的行

### LEFT JOIN（左连接）

```sql
select * from A
left join B
on A.aID = B.bID
```

**结果说明：**
left join是以A表的记录为基础的，A可以看成左表，B可以看成右表，left join是以左表为准的。换句话说，左表(A)的记录将会全部表示出来，而右表(B)只会显示符合搜索条件的记录（例子中为: A.aID = B.bID）。B表记录不足的地方均为NULL。

### RIGHT JOIN（右连接）

```sql
select * from A
right join B
on A.aID = B.bID
```

**结果说明：**
和left join的结果刚好相反，这次是以右表(B)为基础的，A表不足的地方用NULL填充。

### INNER JOIN（内连接）

```sql
select * from A
inner join B
on A.aID = B.bID
```

**结果说明：**
这里只显示出了 A.aID = B.bID 的记录。这说明inner join并不以谁为基础，它只显示符合条件的记录。

### JOIN 语法说明

LEFT JOIN 操作用于在任何的 FROM 子句中，组合来源表的记录。使用 LEFT JOIN 运算来创建一个左边外部联接。左边外部联接将包含了从第一个（左边）开始的两个表中的全部记录，即使在第二个（右边）表中并没有相符值的记录。

**语法：**

```sql
FROM table1 LEFT JOIN table2 ON table1.field1 compopr table2.field2
```

**说明：**
- table1, table2参数用于指定要将记录组合的表的名称
- field1, field2参数指定被联接的字段的名称。且这些字段必须有相同的数据类型及包含相同类型的数据，但它们不需要有相同的名称
- compopr参数指定关系比较运算符："="， "<"， ">"， "<="， ">=" 或 "<>"
- 如果在INNER JOIN操作中要联接包含Memo 数据类型或 OLE Object 数据类型数据的字段，将会发生错误

## ON 与 WHERE 的区别（重要）

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

[src: raw/ingested/2技术/mysql/leetcode-sql.md]
[src: raw/ingested/2技术/mysql/MySQL SQL实战与面试题.md]
[src: raw/ingested/2技术/mysql/MySQL 连接详解.md]

## Related Pages
- [[SQL基础查询]]
- [[SQL子查询与窗口函数]]
- [[SQL数据修改与删除]]
- [[SQL高级DML]]
- [[SQL聚合查询与面试题]]
- [[MySQL连接详解]]

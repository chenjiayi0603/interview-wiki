# SQL 基础查询

## WHERE 基础

### 595. 大的国家

**表：** `World`（name, continent, area, population, gdp），name 为主键。

**题意：** 面积 ≥ 3000000 **或** 人口 ≥ 25000000 的国家，输出 name、population、area。

```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000 OR population >= 25000000;
```

### 1757. 可回收且低脂的产品

**表：** `Products`（product_id, low_fats, recyclable），均为 enum('Y','N')。

**题意：** 既低脂又可回收的 product_id。

```sql
SELECT p.product_id
FROM Products AS p
WHERE p.low_fats = 'Y'
  AND p.recyclable = 'Y';
```

**书写习惯：** 关键字大写、AND/OR 放行首、字段前加表别名。

### 584. 寻找用户推荐人

**表：** `customer`（id, name, referee_id）。

**题意：** 推荐人不是 2 的客户（含 referee_id 为 NULL）。

```sql
-- 写法1：显式处理 NULL
SELECT name FROM customer
WHERE referee_id <> 2 OR referee_id IS NULL;

-- 写法2：IFNULL
SELECT name FROM customer
WHERE IFNULL(referee_id, 0) <> 2;

-- 写法3：NOT IN（需注意 NULL 语义）
SELECT name FROM customer
WHERE id NOT IN (SELECT id FROM customer WHERE referee_id = 2);
```

[src: raw/ingested/2技术/mysql/leetcode-sql.md]

## Related Pages
- [[SQL子查询与窗口函数]]
- [[SQL连接与集合操作]]
- [[SQL数据修改与删除]]

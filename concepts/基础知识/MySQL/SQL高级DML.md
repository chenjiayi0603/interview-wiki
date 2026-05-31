# SQL 高级 DML

## INSERT 高级用法

### INSERT INTO ... SELECT 子查询插入

需求：把一个表某个字段内容复制到另一张表的某个字段

**方案一：指定字段插入**

```sql
Insert into table2(field1,field2,…) select value1,value2,… from table1
```

```sql
insert into table2 (id,a,b,c) select id,a,b,c from table1
```

**表结构相同时可简写：**

```sql
insert into table2 select * from table1
```

**注意事项：**

1. 要求目标表table2必须存在，并且字段field,field2…也必须存在
2. 注意table2的主键约束，如果Table2有主键而且不为空，则 field1， field2…中必须包括主键
3. 注意语法，不要加values，和插入一条数据的sql混了，不要写成：

```sql
-- 错误写法
Insert into Table2(field1,field2,…) values (select value1,value2,… from Table1)
```

### 复制表（批量插入）

**复制表包含数据：**

```sql
create table table3 as select * from table2
```

**只复制表结构到新表：**

```sql
create table table4 as select * from table2 where 1= 2
```

## DELETE 高级用法

### 带子查询的DELETE——删除重复数据

需求：删除重复的数据，只保留最小的那一份数据

**删除重复数据SQL思路：**

1. 先查询到重复数据
2. 再根据table1把相同的字段ID查询出来
3. 过滤最小ID那份数据
4. 再根据表关联级别删除

**第1步：查询重复数据的ID**

```sql
SELECT
    b.id FROM ( SELECT id FROM table1 WHERE a IN 
    ( SELECT a FROM table1 GROUP BY a HAVING count( a ) > 1 ) ) b
```

**第2步：查询每组重复数据中最小的ID**

```sql
select min(id) id from table1 GROUP BY a HAVING count(a) > 1;
```

**第3步：完整的删除语句**

```sql
DELETE 
FROM
	table1 
WHERE
	id IN (
		SELECT
			b.id FROM ( SELECT id FROM table1 WHERE a IN 
			( SELECT a FROM table1 GROUP BY a HAVING count( a ) > 1 ) ) b 
	)
	AND id NOT IN 
	( SELECT id FROM ( SELECT min( id ) id FROM table1 GROUP BY a HAVING count( a ) > 1 ) c )
```

## UPDATE 高级用法

### UPDATE ... JOIN 多表更新

需求：把员工表部门名称字段根据部门表名称进行更新

```sql
update employee e inner join dept d set e.dept_name = d.dept_name where e.dept_id = d.id;
```

[src: raw/ingested/2技术/mysql/MySQL SQL实战与面试题.md]

## Related Pages
- [[SQL基础查询]]
- [[SQL连接与集合操作]]
- [[SQL子查询与窗口函数]]
- [[SQL数据修改与删除]]

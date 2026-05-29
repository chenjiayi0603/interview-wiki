# SQL 聚合查询与面试题

## 聚合查询基础

### 查询平均成绩大于60分学生的学号和平均成绩

```sql
select 学号, avg(成绩)
from score
group by 学号
having avg(成绩)>60;
```

### 查询至少选修两门课程的学生学号

```sql
select 学号, count(课程号) as 选修课程数目
from score
group by 学号
having count(课程号)>=2;
```

### 查询同名同姓学生名单并统计同名人数

```sql
select 姓名,count(*) as 人数
from student
group by 姓名
having count(*)>=2;
```

### 查询不及格的课程并按课程号从大到小排列

```sql
select 课程号
from score 
where 成绩<60
order by 课程号 desc;
```

### 查询每门课程的平均成绩，结果按平均成绩升序排序，平均成绩相同时，按课程号降序排列

```sql
select 课程号, avg(成绩) as 平均成绩
from score
group by 课程号
order by 平均成绩 asc,课程号 desc;
```

### 检索课程编号为"0004"且分数小于60的学生学号，结果按分数降序排列

```sql
select 学号
from score
where 课程号='04' and 成绩 <60
order by 成绩 desc;
```

### 统计每门课程的学生选修人数（超过2人的课程才统计）

要求输出课程号和选修人数，查询结果按人数降序排序，若人数相同，按课程号升序排序

```sql
select 课程号, count(学号) as '选修人数'
from score
group by 课程号
having count(学号)>2
order by count(学号) desc,课程号 asc;
```

### 查询两门以上不及格课程的同学的学号及其平均成绩

```sql
select 学号, avg(成绩) as 平均成绩
from score
where 成绩 <60
group by 学号
having count(课程号)>2;
```

### 查询学生的总成绩并进行排名

```sql
select 学号 ,sum(成绩) from score 
group by 学号
order by sum(成绩);
```

### 查询平均成绩大于60分的学生的学号和平均成绩

```sql
select 学号 ,avg(成绩) from score 
group by 学号  
having avg(成绩 ) >60;
```

## 多表聚合查询

### 查询所有学生的学号、姓名、选课数、总成绩

```sql
select a.学号,a.姓名,count(b.课程号) as 选课数,sum(b.成绩) as 总成绩
from student as a left join score as b
on a.学号 = b.学号
group by a.学号;
```

### 查询平均成绩大于85的所有学生的学号、姓名和平均成绩

```sql
select a.学号,a.姓名, avg(b.成绩) as 平均成绩
from student as a left join score as b
on a.学号 = b.学号
group by a.学号
having avg(b.成绩)>85;
```

### 查询学生的选课情况：学号，姓名，课程号，课程名称

```sql
select a.学号, a.姓名, c.课程号,c.课程名称
from student a inner join score b on a.学号=b.学号
inner join course c on b.课程号=c.课程号;
```

### 查询课程编号为0003且课程成绩在80分以上的学生的学号和姓名

```sql
select a.学号,a.姓名
from student  as a inner join score as b on a.学号=b.学号
where b.课程号='0003' and b.成绩>80;
```

### 查询不同老师所教不同课程平均分从高到低显示（三表连接聚合）

```sql
select a.教师号,a.教师姓名,avg(c.成绩) 
from  teacher as a 
inner join course as b 
on a.教师号= b.教师号 
inner join score  c on b.课程号= c.课程号  
group by a.教师姓名
order by avg(c.成绩) desc;
```

### 查询课程名称为"数学"，且分数低于60的学生姓名和分数（三表连接聚合）

```sql
select a.姓名,b.成绩 
from student as a 
inner join score as b 
on a.学号 =b.学号
inner join course c on b.课程号 =c.课程号
where b.成绩  <60 and c.课程名称 ='数学';
```

### 查询任何一门课程成绩在70分以上的姓名、课程名称和分数（三表连接聚合）

```sql
select a.姓名,c.课程名称 ,b.成绩 
from student as a 
inner join score as b 
on a.学号=b.学号
inner join course c on b.课程号 =c.课程号 
where b.成绩 >70;
```

## SQL 面试题精选

### 查询所有课程成绩小于60分学生的学号、姓名

```sql
select 学号,姓名
from student
where  学号 in (
select 学号 
from score
where 成绩 < 60
);
```

### 查询没有学全所有课的学生的学号、姓名

```sql
select 学号,姓名
from student
where 学号 in(
select 学号 
from score
group by 学号
having count(课程号) < (select count(课程号) from course)
);
```

### 查询出只选修了两门课程的全部学生的学号和姓名

```sql
select 学号,姓名
from student
where 学号 in(
select 学号
from score
group by 学号
having count(课程号)=2
);
```

### 1990年出生的学生名单

```sql
select 学号,姓名 
from student 
where year(出生日期)=1990; 
```

### 查询出每门课程的及格人数和不及格人数（条件聚合 / CASE WHEN）

```sql
select 课程号,
sum(case when 成绩>=60 then 1 
	 else 0 
    end) as 及格人数,
sum(case when 成绩 <  60 then 1 
	 else 0 
    end) as 不及格人数
from score
group by 课程号;
```

[src: raw/ingested/2技术/mysql/MySQL SQL实战与面试题.md]

## Related Pages
- [[SQL基础查询]]
- [[SQL连接与集合操作]]
- [[SQL子查询与窗口函数]]
- [[SQL高级DML]]

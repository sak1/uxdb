# UXDB实例

## 几个窗口函数

聚合函数将结果集进行计算，通常返回一行。 

窗口函数也是基于结果集进行计算，通常返回多行。

使用窗口函数能大幅简化sql代码。

下面介绍几个窗口函数：

### 1、avg()——计算分组数据平均值

创建一张成绩表并插入测试数据，

```sql
create table score(id serial primary key,
	subject character varying(32),
	stu_name character varying(32),
	score numeric(3,0));

insert into score (subject,stu_name,score) values ('chinese','ming',76);
insert into score (subject,stu_name,score) values ('chinese','ma',76);
insert into score (subject,stu_name,score) values ('chinese','tang',81);
insert into score (subject,stu_name,score) values ('english','ma',75);
insert into score (subject,stu_name,score) values ('english','ming',93);
insert into score (subject,stu_name,score) values ('english','tang',65);
insert into score (subject,stu_name,score) values ('math','ming',68);
insert into score (subject,stu_name,score) values ('math','ma',59);
insert into score (subject,stu_name,score) values ('math','tang',58); 
```

查询每名学生学习成绩，显示课程的平均分，通常是先计算出课程的平均分，然后用成绩表与平均分表关联查询，通常SQL比较麻烦，使用窗口函数就很容易实现，例如：

```
select subject,stu_name,score,round(avg(score) over(partition by stu_name),2) from score;

subject | stu_name | score | round 
---------+----------+-------+-------
 english | ma       |    75 | 70.00
 chinese | ma       |    76 | 70.00
 math    | ma       |    59 | 70.00
 chinese | ming     |    67 | 76.00
 english | ming     |    93 | 76.00
 math    | ming     |    68 | 76.00
 math    | tang     |    58 | 68.00
 chinese | tang     |    81 | 68.00
 english | tang     |    65 | 68.00
(9 rows)
```

前三列来源于表score， 第四列表示取每位学员的的平均分，partition by stu_name表示根据字段stu_name进行分组，round(x,y)为四舍五入函数。



### 2.row_number() ——对结果集分组数据标注行号

例如：

```sql
/*如分学科成绩排名*/
select row_number() over (partition by subject order by score desc) ,* from score; 
row_number | id | subject | stu_name | score 
------------+----+---------+----------+-------
          1 |  3 | chinese | tang     |    81
          2 |  2 | chinese | ma       |    76
          3 |  1 | chinese | ming     |    76
          1 |  5 | english | ming     |    93
          2 |  4 | english | ma       |    75
          3 |  6 | english | tang     |    65
          1 |  7 | math    | ming     |    68
          2 |  8 | math    | ma       |    59
          3 |  9 | math    | tang     |    58
(9 rows)
```

row_number()窗口函数显示的是分组后记录的行号，不指定partition属性，函数显示表所有记录的行号，例如：

```sql
/*如分学科成绩不做排名*/
select row_number() over (order by id) as rownum,* from score;
 rownum | id | subject | stu_name | score 
--------+----+---------+----------+-------
      1 |  1 | chinese | ming     |    76
      2 |  2 | chinese | ma       |    76
      3 |  3 | chinese | tang     |    81
      4 |  4 | english | ma       |    75
      5 |  5 | english | ming     |    93
      6 |  6 | english | tang     |    65
      7 |  7 | math    | ming     |    68
      8 |  8 | math    | ma       |    59
      9 |  9 | math    | tang     |    58
(9 rows)
```

### rank()——特定的对结果集分组数据分等级标注行号

和row_number()相似，区别在于当组内某行字段值相同时，行号重复且行号可不连续。例如：

```
select rank() over (partition by subject order by score),* from score;
 rank | id | subject | stu_name | score 
------+----+---------+----------+-------
    1 |  2 | chinese | ma       |    76
    1 |  1 | chinese | ming     |    76
    3 |  3 | chinese | tang     |    81
    1 |  6 | english | tang     |    65
    2 |  4 | english | ma       |    75
    3 |  5 | english | ming     |    93
    1 |  9 | math    | tang     |    58
    2 |  8 | math    | ma       |    59
    3 |  7 | math    | ming     |    68
(9 rows)
```

### dense_rank  ()

dense_rank （）窗口函数和rank（）窗口函数相似，主要区别为当组内某行字段值相同时，虽然行号重复，行号连续的，如下所示：

```sql
select dense_rank() over (partition by subject order by score),* from score;

 dense_rank | id | subject | stu_name | score 
------------+----+---------+----------+-------
          1 |  2 | chinese | ma       |    76
          1 |  1 | chinese | ming     |    76
          2 |  3 | chinese | tang     |    81
          1 |  6 | english | tang     |    65
          2 |  4 | english | ma       |    75
          3 |  5 | english | ming     |    93
          1 |  9 | math    | tang     |    58
          2 |  8 | math    | ma       |    59
          3 |  7 | math    | ming     |    68
(9 rows)
```

percent_rank()

### lag()——获取行偏移offset那行某个字段的数据

```sql
select lag(id,1)over(),* from score; 

 lag | id | subject | stu_name | score 
-----+----+---------+----------+-------
     |  2 | chinese | ma       |    76
   2 |  3 | chinese | tang     |    81
   3 |  4 | english | ma       |    75
   4 |  5 | english | ming     |    93
   5 |  6 | english | tang     |    65
   6 |  7 | math    | ming     |    68
   7 |  8 | math    | ma       |    59
   8 |  9 | math    | tang     |    58
   9 |  1 | chinese | ming     |    76
(9 rows)
```



### first_ value() ——取结果集每个分组第一行数据的字段值

例如：score表按课程分组后取分组的第一行的分数，例如：

```
select first_value(score) over(partition by subject),* from score; 
 first_value | id | subject | stu_name | score 
-------------+----+---------+----------+-------
          76 |  2 | chinese | ma       |    76
          76 |  3 | chinese | tang     |    81
          76 |  1 | chinese | ming     |    76
          93 |  5 | english | ming     |    93
          93 |  6 | english | tang     |    65
          93 |  4 | english | ma       |    75
          68 |  7 | math    | ming     |    68
          68 |  8 | math    | ma       |    59
          68 |  9 | math    | tang     |    58
(9 rows)
```

### last_value () 取结果集每个分组的最后一行数据的字段值。

例如score表按课程分组后取分组的最后一行的分数，例如：

```sql
select last_value(score) over(partition by subject),* from score;
 last_value | id | subject | stu_name | score 
------------+----+---------+----------+-------
         76 |  2 | chinese | ma       |    76
         76 |  3 | chinese | tang     |    81
         76 |  1 | chinese | ming     |    76
         75 |  5 | english | ming     |    93
         75 |  6 | english | tang     |    65
         75 |  4 | english | ma       |    75
         58 |  7 | math    | ming     |    68
         58 |  8 | math    | ma       |    59
         58 |  9 | math    | tang     |    58
(9 rows)
```



### nth_value()—— 取结果集每个分组的指定行数据的字段值

例如score表按课程分组后取分组的第二行的分数，例如：

```sql
select nth_value(score,1) over(partition by subject) ,* from score ; 

 nth_value | id | subject | stu_name | score 
-----------+----+---------+----------+-------
        76 |  2 | chinese | ma       |    76
        76 |  3 | chinese | tang     |    81
        76 |  1 | chinese | ming     |    76
        93 |  5 | english | ming     |    93
        93 |  6 | english | tang     |    65
        93 |  4 | english | ma       |    75
        68 |  7 | math    | ming     |    68
        68 |  8 | math    | ma       |    59
        68 |  9 | math    | tang     |    58
(9 rows)
/*nth value (字段名,第n行) */
```

### 窗口函数别名

如果SQL中多次使用某个窗口函数，可以别名化使用， 语法如下：

```sql
select round(avg(score) over(r),2) as avg , sum(score) over(r), * from score window r as (partition by subject); 
avg  | sum | id | subject | stu_name | score 
-------+-----+----+---------+----------+-------
 77.67 | 233 |  2 | chinese | ma       |    76
 77.67 | 233 |  3 | chinese | tang     |    81
 77.67 | 233 |  1 | chinese | ming     |    76
 77.67 | 233 |  5 | english | ming     |    93
 77.67 | 233 |  6 | english | tang     |    65
 77.67 | 233 |  4 | english | ma       |    75
 61.67 | 185 |  7 | math    | ming     |    68
 61.67 | 185 |  8 | math    | ma       |    59
 61.67 | 185 |  9 | math    | tang     |    58
 (9 rows)
 
 /*window r as (partition by subject) 翻译一下就是 partition by subject = r，很好理解，是吧 */
```


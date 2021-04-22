# UXDB实例

## 聚合函数

聚合函数是指对结果集进行计算的函数，聚合函数的出现，大大简化了统计等工作，是SQL编写中的重要常识；

聚合函数有哪些呢？常见的有avg()、 sum()、 min()、 max()、count()等，这里介绍特殊的字符串聚合函数。

### 需求：有如下数据，需要连成“中国 西安,襄阳,喀什”等

```
create table city (country character varying(64),city character varying(64));
insert into city values('中国','西安');
insert into city values('中国','襄阳');
insert into city values('中国','喀什');
insert into city values('蒙古','乌兰巴托');
insert into city values('蒙古','苏赫巴托尔');
```

### string_agg()——将结果集某个字段的所有行连接成字符串

将city字段连接成字符串：

```
select country,string_agg(city,',') from city group by country; 
 country |     string_agg      
---------+---------------------
 中国    | 西安,襄阳,喀什
 蒙古    | 乌兰巴托,苏赫巴托尔
(2 rows)
```

### array_agg函数——功能同上，返回的类型为数组

和string_agg函数类似，最主要的区别为返回的类型为数组，数组数据类型同输入参数数据类型一致

```
select country, array_agg(city) from city group by country; 
 country |       array_agg       
---------+-----------------------
 中国    | {西安,襄阳,喀什}
 蒙古    | {乌兰巴托,苏赫巴托尔}
(2 rows)
```

array_agg函数优点在于可以使用数组相关函数和操作符

### mode函数——取分组中出现频率最高的值或表达式, 如果高频值有多个, 就随机取一个

示例：

```
create table testmode(id int, info text);
insert into testmode values (1,'test1');
insert into testmode values (1,'test1');
insert into testmode values (1,'test2');
insert into testmode values (1,'test3');
insert into testmode values (2,'test1');
insert into testmode values (2,'test1');
insert into testmode values (2,'test1');
insert into testmode values (3,'test4');
insert into testmode values (3,'test4');
insert into testmode values (3,'test4');
insert into testmode values (3,'test4');
insert into testmode values (3,'test4');
```

取出所有数据中, 出现频率最高的info, 有可能是test1，也有可能是test4

```sql
select mode() within group (order by info) from testmode;
 mode  
 test1
(1 row)
```

按id来分组, 取出组内出现频率最高的info值

```sql
select mode() within group (order by info) from testmode group by id;
 mode  
 test1
 test1
 test4
(3 rows)
```



### 还有哪些聚合函数呢？

```sql
SELECT DISTINCT(proname) FROM ux_proc WHERE proisagg order by proname;
```

array_agg
avg
bit_and
bit_or
bool_and
bool_or
corr
count
covar_pop
covar_samp
cume_dist
dense_rank
every
json_agg
json_object_agg
jsonb_agg
jsonb_object_agg
max
min
mode
percent_rank
percentile_cont
percentile_disc
rank
regr_avgx
regr_avgy
regr_count
regr_intercept
regr_r2
regr_slope
regr_sxx
regr_sxy
regr_syy
stddev
stddev_pop
stddev_samp
string_agg
sum
var_pop
var_samp
variance
xmlagg

数据库设计、SQL编写过程中， 需要注意函数与数据的配合，还要规避函数应用的死角，比如对空数据的处理等等，需要认真运筹，避免由于带来的应用故障。
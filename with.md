# UXDB实例

## 使用WITH查询实现多级节点的递归查询

> WITH查询是UXDB高级SQL特性之一，常称为CTE(Common Table Expressions), 可以在复杂查询中定义一个辅助语句（可理解成一个临时表），常用于复杂查询或递归查询应用场景。可以简化SQL代码，减少SQL嵌套层数，提高SQL代码的可读性。辅助语句只需要计算一次即可在主查询中多次使用；在不需要共享查询结果时，相比视图更轻量。

### 我们看一个简单的例子

```sql
with t as(select generate_series(1,3)) 
select * from t; 
执行结果如下：
generate_series 
-----------------------
         1
         2
         3
(3 rows) 
```

本例中，聚合函数generate_series(1,3)生成一个从1到3，步长为1的数值序列，这里定义了一条辅助语句t(类临时表)用于取数，用于在之后的主查询语句中查询t。

### 递归示例

CTE其重要属性是递归（RECURSIVE）,使用递归可以引用自己的输出，可用于层次结构或树状结构的应用场景，例如针对如下数据，要求输出递归结果：

| id   | name     | fatherid |
| ---- | -------- | -------- |
| 1    | 中国     | 0        |
| 2    | 黑龙江省 | 1        |
| 3    | 广东省   | 1        |
| 4    | 哈尔滨市 | 2        |
| 5    | 广州市   | 3        |
| 6    | 汕头市   | 3        |
| 7    | 南岗区   | 4        |
| 8    | 清河区   | 6        |

当给定一个id时能得到它完整的地名，如当id=7时，地名是：中国黑龙江省哈尔滨市南岗区，当id=5时，地名是： 中国广东省广州市，这是一个典型的层次数据递归应用场景，可以通过UXDB的WITH查询实现，首先我们创建address表并插入数据，如下所示：

```sql
create table address(id int4, name varchar(32) ,fatherid int4); 
insert into address values(1,'中国',0); 
insert into address values(2,'黑龙江省',1); 
insert into address values(3,'广东省',1); 
insert into address values(4,'哈尔滨市',2); 
insert into address values(5,'广州市',3); 
insert into address values(6,'汕头市',3); 
insert into address values(7,'南岗区',4);
insert into address values(8,'清河区',6);
```

使用WITH查询，检索ID为7以及以上的所有父节点，如下所示：

```sql
with recursive r as( 
select * from address where id=7 
UNION ALL 
select address.* from address,r where address.id = r.fatherid)
select * from r order by id; 

*union all 用于合并两个或多个SELECT语句的结果集，允许重复
```

```sql
查询结果如下：
 id |  name  | fatherid 
----+----------+----------
 1 | 中国   |    0
 2 | 黑龙江省 |    1
 4 | 哈尔滨市 |    2
 7 | 南岗区  |    4
(4 rows)
```

查询结果是ID=7节点以及它的所有父节点，下面我们把name字段输出结果合并成“中国黑龙江省哈尔滨市南岗区”，可通过string_agg函数实现，string_agg是把字符合并起来的聚合函数如下所示：

```sql
with recursive r as( select * from address where id=7 
UNION ALL 
select address.* from address , r where address .id= r. fatherid)
select string_agg(name ,'') from (select name from r order by id) n; 
结果：
string_agg
------------------------
中国黑龙江省哈尔滨市南岗区
(1 row)
```

以上是查找当前节点以及当前节点的所有父节点，也可以查找当前节点以及其下的所有子节点，更改where条件，如查找哈尔滨市及管辖的区，代码如下所示。

```sql
with recursive r as( 
select * from address where id = 4 
union all 
select address.* from address, r where address.fatherid = r.id)
select * from r order by id;   *union all 用于合并两个或多个SELECT语句的结果集，允许重复


id |   name   | fatherid 
----+----------+----------
  4 | 哈尔滨市 |        2
  7 | 南岗区   |       4
(2 rows)

```
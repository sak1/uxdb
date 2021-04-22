# UXDB实例

## 行锁定

uxrowlocks函数来显示指定表的行锁定信息。通常仅限于超级用户使用

使用uxrowlocks模块前，首先需要执行CREATE EXTENSION命令：

### 引入

```sql
CREATE EXTENSION uxrowlocks;
CREATE EXTENSION
select * from uxrowlocks('test');
locked_row | locker | multi | xids | modes | pids 
------------+--------+-------+------+-------+------
(0 rows)
```

参数是一个表名。结果是一个记录集合，其中每一行对应表中一个被锁定的行。输出列如下表

### uxrowlocks输出列说明

| 名称       | 类型      | 描述                                                         |
| ---------- | --------- | ------------------------------------------------------------ |
| locked_row | tid       | 被锁定行的元组ID（TID）                                      |
| locker     | xid       | 持锁者的事务ID，如果是多事务则为多事务ID。                   |
| multi      | boolean   | 如果持锁者是一个多事务，则为真。                             |
| xids       | xid[]     | 持锁者的事务ID（如果是多事务则多于一个）。                   |
| lock_type  | text[]    | 持锁者的锁模式（如果是多事务则多于一个），是一个Key Share、Share、For No Key Update、No Key Update、For \|Update、Update组成的数组。 |
| pids       | integer[] | 锁定后端的进程ID（如果是多事务则多于一个）                   |

ux_stat_scan_tables角色的成员和在该表上拥有SELECT权限的用户。

uxrowlocks会为目标表加AccessShareLock并读取每一行来收集行的锁定信息，对于大表来说速度较慢。

uxrowlocks不显示被锁定行的内容。

### 查看被锁定行的内容，例如：

```sql
SELECT * FROM test AS a, uxrowlocks('test') AS p
  WHERE p.locked_row = a.ctid;
   id | testid | personid | imgurl | rectime | locked_row | locker | multi | xids | modes | pids 
----+--------+----------+--------+---------+------------+--------+-------+------+-------+------
(0 rows)
```

```sql

```


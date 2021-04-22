# UXDB实例

## 回滚恢复truncate数据

事务的一个高级功能就是它能够通过预写日志设计来执行事务性的DDL。就是把DDL语句放在一个事务中，比如TRUNCATE表。

### 模拟数据

```shell
./uxbench -M prepared -n -r -P 1 -f ./test.sql -c 50 -j 50 -T 300
```

### 登录数据库

```shell
uxsql
```

### 看看有多少数据

```sql
select count(*) from test; 
  count  
 3041060
(1 row)
```

### 开始事务

```sql
uxdb=# BEGIN;
BEGIN
```

### 清除数据

```sql
uxdb=# truncate test;
TRUNCATE TABLE
```

### 观察下，数据没有了

```sql
select count(*) from test ; 
 count 
     0
(1 row)
```

### 回滚

```sql
rollback;
ROLLBACK
```

### 观察下

```sql
select count(*) from test ; 
  count  
 3041060
(1 row)
```

表中的数据都完好无损，和回收站的功能差不多，是吧！
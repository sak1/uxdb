# UXDB实例

## 常见的库表操作

### 建库
~~~sql
create database abc;
~~~
### 连接库
~~~sql
\c abc;
~~~
### 列表名
~~~sql
select tablename from ux_tables where schemaname='public';
\dt
~~~
### 建表
~~~sql
CREATE TABLE user_tbl(name VARCHAR(20), signup_date DATE);
~~~
### 插入数据
~~~sql
INSERT INTO user_tbl(name, signup_date) VALUES('张三', '2013-12-22');
~~~
### 查询数据
~~~sql
SELECT * FROM user_tbl;
~~~
### 更新数据

```sql
UPDATE user_tbl set name = '李四' WHERE name = '张三';
```

### 删除记录
```sql
DELETE FROM user_tbl WHERE name = '李四' ;
```

### 表结构修改

~~~sql
ALTER TABLE user_tbl ALTER COLUMN signup_date SET NOT NULL;
~~~
### 列添加

```sql
ALTER TABLE user_tbl ADD email VARCHAR(40);
```

### 列更名
```sql
ALTER TABLE user_tbl RENAME COLUMN signup_date TO signup;
```

### 表更名

```sql
ALTER TABLE user_tbl RENAME TO backup_tbl;
```

#### 删除表
```sql
DROP TABLE IF EXISTS backup_tbl;
```

***
### 索引

#### 建索引

~~~
create index name_index ON user_tbl(name);
/*name为列名*/
~~~
#### 重建索引
~~~sql
reindex index name_index;
~~~
### 删除索引

```sql
drop index name_index;
```

***
### 存储过程
#### 先建表

~~~sql
create table tb1 (a integer, b integer);
~~~
#### 建存储过程：

~~~sql
CREATE PROCEDURE insert_data(a integer,b integer) LANGUAGE SQL
AS $$
INSERT INTO tb1 VALUES(a);
INSERT INTO tb1 VALUES(b);
$$;
~~~

#### 调用存储过程
```sql
CALL insert_data(1, 2);
```

#### 查看存储过程
```sql
select * from ux_proc;
```

------

### 扩展插件

#### 安装扩展

```sql
CREATE EXTENSION extensions_name;
```

#### 查询扩展安装

```sql
select * from ux_available_extensions;
```


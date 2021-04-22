# UXDB实例 

列存功能在V2.1.0.4及以上版中，这里我们做介绍性说明。

## 列存的用法

### 什么情况下用到列存？

- 记录条数超过百万、千万甚至上亿条
- 关键业务以查询为主
- 查询涉及的列数占总列数的10%左右

### 初始化集群

```shell
./initdb -D column -W 
Enter new superuser password: 
Enter it again: 
/*输入一个新的管理员口令*/
```

### 启动数据库服务

```
./ux_ctl start -D ../column 
```

### 连接数据库

```
./uxsql -d uxdb 
Password: 
/*输入初始化时设置的口令 */
```

### 创建列存表

例如：

```sql
create table tablec(id int, name text, age int) with(orientation=column, appendonly=true);
```

相对于传统的行存储多了with子句部分；增、删、改、查语句与行存储表的使用方式相同。

### DML操作

```sql
/*插入*/
INSERT INTO table_name VALUES(1,'zhangsan1',23);
/*删除*/
DELETE FROM table_name WHERE id=1;
/*更改*/
UPDATE table_name SET name='lisi' WHERE id=2;
/*查找*/
SELECT * FROM table_name;
SELECT name,age FROM table_name WHERE id=3;
```

### 修改列存表

```
//增加列
ALTER TABLE table_name ADD COLUMN address text;
//删除列
ALTER TABLE table_name DROP COLUMN address;
```

### 创建约束

```
CREATE TABLE products(product_no int, name text, price numeric CONSTRAINT positive_price CHECK(price > 0)) WITH (ORIENTATION=column, APPENDONLY=true);
```

### 删除表数据和表定义

```
DROP TABLE table_name;
```

### 创建带数据压缩的列存表

```
create table tbl_col_compressed(id int, name text, age int) with (orientation=column,appendonly=true, compresstype=zlib, compresslevel=5);
```

compresstype配置压缩算法，可配置zlib和RLE_TYPE；compresslevel表示压缩算法对于的压缩级别，当compresstype配置zlib时取值1~9，当compresstype配置RLE_TYPE时，取值1~4，数值越大压缩效率越高，cpu消耗较大。
表数据存储采用压缩方式，节省磁盘存储空间，但是在数据写入、读取过程会引入压缩、解压缩的CPU损耗。
查看表数据占用存储空间：

```
select ux_relation_size('tbl_col_compressed');
```

### 导入/导出表数据

从文本文件导入表数据：

```
CREATE TABLE column_copy (id int,name text,age int) WITH(orientation=column,appendonly=true);
COPY column_copy FROM '/home/uxdb/uxdbinstall/dbsql/bin/copy_from.txt' (DELIMITER ' ');
```

copy_from.txt文件内容示例：
1 wanger 23
2 zhangsan 25
3 lisi 28
将表数据导出到文本文件：

```
COPY column_copy TO '/home/uxdb/uxdbinstall/dbsql/bin/copy_to.txt';
```

### VACUUM清理表数据

在对表执行删除、修改后，表数据文件里会残留老旧的记录值，为了将这些老旧记录值删
掉，可执行vacuum操作。uxsql窗口执行如下数据库命令：

```sql
vacuum FULL table_name;
```

### 索引操作

为了加速特定列上的条件查询速度，可在特定列上创建索引。

```sql
CREATE INDEX index_name ON table_name(id);
\di index_name
DROP INDEX index_name;
```

### brin粗粒度索引

```sql
/*创建brin粗粒度索引*/
create index index_name ON table_name using brin(column_name) with(pages_per_range=128);
```

```
/*查看表和索引定义*/
\d table_name
```

128表示128个块对于一个索引条目，一个块包含128表记录行。pages_per_range小，索引越精细，索引文件里索引行记录越多，占用空间越大。

### Partition表分区

#### 范围分区

列存表被分区到由键列或列集定义的“范围”中，分配给不同分区的值范围之间没有重叠。
例如，可以按日期范围进行分区， 也可以按特定业务对象的标识符范围进行分区。

• 创建列存分区表

```sql
CREATE TABLE column01 (id int,name text,age int) PARTITION BY RANGE(id)
 WITH(appendonly=true,orientation=column);
```

parttition by指定分区方式和分区键。

• 创建分区，以创建5个分区子表为例

```sql
CREATE TABLE column01_1000 PARTITION OF column01 FOR VALUES FROM('1')
 TO('1000') WITH(appendonly=true,orientation=column);
CREATE TABLE column01_2000 PARTITION OF column01 FOR VALUES FROM('1000')
 TO('2000') WITH(appendonly=true,orientation=column);
CREATE TABLE column01_3000 PARTITION OF column01 FOR VALUES FROM('2000')
 TO('3000') WITH(appendonly=true,orientation=column);
CREATE TABLE column01_4000 PARTITION OF column01 FOR VALUES FROM('3000')
 TO('4000') WITH(appendonly=true,orientation=column);
CREATE TABLE column01_5000 PARTITION OF column01 FOR VALUES FROM('4000')
 TO('5000') WITH(appendonly=true,orientation=column);
```

• 查看分区子表

```sql
\d column01_1000
```

• 插入数据

```sql
insert into column01 values (generate_series(1,4999),'name1',34);
```

• 查看分区表数据

```sql
EXPLAIN select * from column01;
```

• 删除分区子表

```sql
DROP TABLE column01_1000;
```

### 列表分区

列存表通过明确列出每个分区中出现的键值进行分区。

#### • 创建分区主表

```sql
create table cs_list_part (id int8,random_char varchar(100),day_id varchar(8)) PARTITION BY
 LIST(day_id) WITH(appendonly=true,orientation=column);
```

#### • 创建分区从表

```sql
CREATE TABLE cs_list_part_p20201130 PARTITION OF cs_list_part FOR VALUES in
 ('20201130');
CREATE TABLE cs_list_part_p20201201 PARTITION OF cs_list_part FOR VALUES in
 ('20201201');
CREATE TABLE cs_list_part_p20201202 PARTITION OF cs_list_part FOR VALUES in
 ('20201202');
CREATE TABLE cs_list_part_p20201203 PARTITION OF cs_list_part FOR VALUES in
 ('20201203');
```

#### • 插入数据

```sql
insert into cs_list_part select * from (
 select generate_series(1, 5) as id, md5(random()::text) as info , '20201130' as day_id union all
 select generate_series(1, 5) as id, md5(random()::text) as info , '20201201' as day_id union all
 select generate_series(1, 5) as id, md5(random()::text) as info , '20201202' as day_id union all
 select generate_series(1, 5) as id, md5(random()::text) as info , '20201203' as day_id 
) t0;
```

#### 5 查询分区表数据

分区主表

```sql
select * from cs_list_part order by day_id,id;
```

分区从表

```sql
select * from cs_list_part_p20201130;
```

• 使用不存在的分区值20201129插入记录，报错

```sql
insert into cs_list_part select * from (
 select generate_series(1, 5) as id, md5(random()::text) as info , '20201129' as day_id 
) t0;
ERROR:  no partition of relation "cs_list_part" found for row
DETAIL:  Partition key of the failing row contains (day_id) = (20201129).
```



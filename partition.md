# UXDB实例

## 分区表详解

uxdb支持两种分区表，传统分区表与内置分区表
分区表的优势主要体现在降低大表管理成本和某些场景的性能提升，传统分区表是通过继承和触发器方式实现的，我们先了解下继承表：

### 继承表

#### 创建父表、子表

```sql
create table  tbl_log(id int4, create_date date,log_type text) ; 
create table tbl_log_sql(sql text) inherits(tbl_log); 
/*nherits(tbl_log）表示表tbl_log_sql继承表tbl_log*/
```

子表定义了额外字段sql， 其他字段继承父表tbl_log， 查看tbl_log_sql表结构：

```sql
\d tbl_log_sql
              Table "public.tbl_log_sql"
   Column    |  Type   | Collation | Nullable | Default 
-------------+---------+-----------+----------+---------
 id          | integer |           |          | 
 create_date | date    |           |          | 
 log_type    | text    |           |          | 
 sql         | text    |           |          | 
Inherits: tbl_log  
/*表示继承了表tbl_log*/
```

子表比父表多了一个字段

#### 子表父表数据对比

父表和子表都可以插入数据，接着分别在父表和子表中插入一条数据：

```
insert into tbl_log values(1,'2020-08-26',null);
INSERT 0 1
uxdb=# select * from tbl_log_sql;
 id | create_date | log_type | sql 
----+-------------+----------+-----
(0 rows)
```

父表插入数据，子表不变；

```
insert into tbl_log_sql values(2,'2020-08-27',null,'select version()');
select * from tbl_log;
 id | create_date | log_type 
----+-------------+----------
  1 | 2020-08-26  | 
  2 | 2020-08-27  | 
(2 rows)
```

子表插入数据库，父表随之变化；——“知子莫若父”啊，反之不成。

#### 确定数据来源

查看表oid：

```
select tableoid,* from tbl_log ; 
 tableoid | id | create_date | log_type 
----------+----+-------------+----------
    91336 |  1 | 2017-08-26  | 
    91342 |  2 | 2020-08-27  | 
(2 rows)
```

#### 找表名：

```
select p.relname,c.* from tbl_log c,ux_class p where c.tableoid =p.oid;
  relname   | id | create_date | log_type 
-------------+----+-------------+----------
 tbl_log     |  1 | 2017-08-26  | 
 tbl_log_sql |  2 | 2020-08-27  | 
(2 rows)
```

#### select父表

在父表名称前加上only：

```
select * from only tbl_log; 
 id | create_date | log_type 
----+-------------+----------
  1 | 2017-08-26  | 
(1 row)
```

#### update、 delete父表

如果父表名称前没有加only， 则会对父表和所有子表进行操作

```sql
delete from tbl_log; 
select count(*)  from tbl_log; 
 count 
-------
     0
(1 row)
select * from tbl_log_sql;
 id | create_date | log_type | sql 
----+-------------+----------+-----
(0 rows)
```

都被删除了吧。
对于使用了继承表的的update、delete的操作需特别谨慎，小心数据被连带删除。您可以使用以下命令查看是否有继承表。

#### 列出所有父表

```sql
SELECT
	nspname ,
	relname ,
	COUNT(*) AS partition_num
FROM	
	ux_class c ,
	ux_namespace n ,
	ux_inherits i
WHERE
	c.oid = i.inhparent
	AND c.relnamespace = n.oid
	AND c.relhassubclass
	AND c.relkind = 'r'
GROUP BY 1,2 ORDER BY partition_num DESC;
nspname | relname | partition_num 
---------+---------+---------------
 public  | tbl_log |             1
(1 row)
```

### 传统分区表

#### 创建传统分区表

第一步：创建父表，如果父表上定义了约束，子表会继承，因此除非是全局约束，否则不应该在父表上定义约束，另外，父表不应该写入数据。

```sql
create table log_ins(id serial,user_id int4,create_time timestamp(0) without time zone); 
```

第二步：创建子表(继承表或分区)，字段定义与父表保持一致。13个表

```
create table log_ins_history(check(create_time <'2020-01-01')) inherits (log_ins);
create table log_ins_202001(check(create_time >='2020-01-01' and create_time <'2020-02-01')) inherits (log_ins);
create table log_ins_202002(check(create_time >='2020-02-01' and create_time <'2020-03-01')) inherits (log_ins);
create table log_ins_202003(check(create_time >='2020-03-01' and create_time <'2020-04-01')) inherits (log_ins);
create table log_ins_202004(check(create_time >='2020-04-01' and create_time <'2020-05-01')) inherits (log_ins);
create table log_ins_202005(check(create_time >='2020-05-01' and create_time <'2020-06-01')) inherits (log_ins);
create table log_ins_202006(check(create_time >='2020-06-01' and create_time <'2020-07-01')) inherits (log_ins);
create table log_ins_202007(check(create_time >='2020-07-01' and create_time <'2020-08-01')) inherits (log_ins);
create table log_ins_202008(check(create_time >='2020-08-01' and create_time <'2020-09-01')) inherits (log_ins);
create table log_ins_202009(check(create_time >='2020-09-01' and create_time <'2020-10-01')) inherits (log_ins);
create table log_ins_202010(check(create_time >='2020-10-01' and create_time <'2020-11-01')) inherits (log_ins);
create table log_ins_202011(check(create_time >='2020-11-01' and create_time <'2020-12-01')) inherits (log_ins);
create table log_ins_202012(check(create_time >='2020-12-01' and create_time <='2020-12-31')) inherits (log_ins);
```

第三步：子表创建约束，满足约束条件的数据才能写入对应子表，子表约束值范围不要有重叠。
第四步：子表创建索引，子表不继承父表索引，子表索引需要手工创建。

```
CREATE INDEX idx_his_ctime ON log_ins_history USING btree(create_time);
CREATE INDEX idx_log_ins_202001_ctime ON log_ins_202001 USING btree(create_time); 
CREATE INDEX idx_log_ins_202002_ctime ON log_ins_202002 USING btree(create_time); 
CREATE INDEX idx_log_ins_202003_ctime ON log_ins_202003 USING btree(create_time); 
CREATE INDEX idx_log_ins_202004_ctime ON log_ins_202004 USING btree(create_time); 
CREATE INDEX idx_log_ins_202005_ctime ON log_ins_202005 USING btree(create_time); 
CREATE INDEX idx_log_ins_202006_ctime ON log_ins_202006 USING btree(create_time); 
CREATE INDEX idx_log_ins_202007_ctime ON log_ins_202007 USING btree(create_time); 
CREATE INDEX idx_log_ins_202008_ctime ON log_ins_202008 USING btree(create_time); 
CREATE INDEX idx_log_ins_202009_ctime ON log_ins_202009 USING btree(create_time); 
CREATE INDEX idx_log_ins_202010_ctime ON log_ins_202010 USING btree(create_time); 
CREATE INDEX idx_log_ins_202011_ctime ON log_ins_202011 USING btree(create_time); 
CREATE INDEX idx_log_ins_202012_ctime ON log_ins_202012 USING btree(create_time); 
```

第五步：[可选] 在父表上定义INSERT、 DELETE、 UPDATE触发器，将SQL分发到对应子表，因为应用可以根据分区规则定位到对应分区进行DML操作。

父表上不存储数据，可以不用在父表上创建索引。

创建触发器函数，设置数据插入父表时的路由规则，

```
CREATE OR REPLACE FUNCTION log_ins_insert_trigger()
RETURNS trigger 
LANGUAGE pluxsql 
AS $function$
BEGIN
IF (NEW.create_time <'2020-01-01') THEN 
	INSERT INTO log_ins_history VALUES (NEW.*); 
	ELSIF (NEW.create_time>='2020-01-01' and NEW.create_time <'2020-02-01') THEN 
		INSERT INTO log_ins_202001 VALUES (NEW.*); 
	ELSIF (NEW.create_time>='2020-02-01' and NEW.create_time<'2020-03-01') THEN 
		INSERT INTO log_ins_202002 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-03-01' and NEW.create_time<'2020-04-01') THEN 
		INSERT INTO log_ins_202003 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-04-01' and NEW.create_time<'2020-05-01') THEN 
		INSERT INTO log_ins_202004 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-05-01' and NEW.create_time<'2020-06-01') THEN 
		INSERT INTO log_ins_202005 VALUES (NEW.*);	
	ELSIF (NEW.create_time>='2020-06-01' and NEW.create_time<'2020-07-01') THEN 
		INSERT INTO log_ins_202006 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-07-01' and NEW.create_time<'2020-08-01') THEN 
		INSERT INTO log_ins_202007 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-08-01' and NEW.create_time<'2020-09-01') THEN 
		INSERT INTO log_ins_202008 VALUES (NEW.*);	
	ELSIF (NEW.create_time>='2020-09-01' and NEW.create_time<'2020-10-01') THEN 
		INSERT INTO log_ins_202009 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-10-01' and NEW.create_time<'2020-11-01') THEN 
		INSERT INTO log_ins_202012 VALUES (NEW.*);
	ELSIF (NEW.create_time>='2020-11-01' and NEW.create_time<'2020-12-01') THEN 
		INSERT INTO log_ins_202011 VALUES (NEW.*);			
	ELSIF (NEW.create_time >='2020-12-01' and NEW.create_time <='2020-12-31') THEN 
		INSERT INTO log_ins_202012 VALUES (NEW.*); 
	ELSE
		RAISE EXCEPTION 'create_time out of range. Fix the log_ins_insert_trigger() function'; 
	END IF; 
	RETURN NULL; 
END; 
$function$; 
```



在父表上定义插入触发器

```
CREATE TRIGGER insert_log_ins_trigger BEFORE INSERT ON log_ins FOR EACH ROW 
EXECUTE PROCEDURE log_ins_insert_trigger(); 
```

第六步：设置constraint_exclusion参数设为on，如果为off， 父表上的SQL性能会降低。

内置分区表执行计划受constraint exclusion参数影响，关闭此参数后，根据分区键查询时执行计划不会定位到相应分区。 先在会话级关闭此参数。

constraint_exclusion是表分区策略，意味着检查**约束**放在了表上，以限制可以在表中插入哪种数据。

```sql
SET constraint_exclusion  =off; 
```


不建议设置成on，优化器通过检查约束来优化查询的方式本身就带来一定开销，如果所有表都启用这个特性，会加重优化器负担。

触发器创建完成后，往父表log_ins插人数据时，会执行触发器并触发函数log_ins_insert_trigger()表数据插入到相应分区中。 DELETE、 UPDATE触发器和函数创建过程和INSERT方式类似，不再列出，这步完成之后，传统分区表的创建步骤已全部完成。

#### 使用分区表

插入数据库

```sql
INSERT INTO log_ins(user_id,create_time) 
SELECT round(1000000000*random()),generate_series('2019-12-01'::date,'2020-12-31'::date,'1 minute'); 
```

查询数据

```
select count(*) from log_ins;
 count  
--------
 570241
(1 row)
select * from only log_ins;
 id | user_id | create_time 
----+---------+-------------
(0 rows)
select count(*) from  log_ins_202001;
 count 
-------
 44640
(1 row)
```

查看各表情况

```
\dt+  log_ins* 
                        List of relations
 Schema |      Name       | Type  | Owner |  Size   | Description 
--------+-----------------+-------+-------+---------+-------------
 public | log_ins         | table | uxdb  | 0 bytes | 
 public | log_ins_202001  | table | uxdb  | 2016 kB | 
 public | log_ins_202002  | table | uxdb  | 1920 kB | 
 public | log_ins_202003  | table | uxdb  | 2016 kB | 
 public | log_ins_202004  | table | uxdb  | 1984 kB | 
 public | log_ins_202005  | table | uxdb  | 2016 kB | 
 public | log_ins_202006  | table | uxdb  | 1984 kB | 
 public | log_ins_202007  | table | uxdb  | 2016 kB | 
 public | log_ins_202008  | table | uxdb  | 2016 kB | 
 public | log_ins_202009  | table | uxdb  | 1984 kB | 
 public | log_ins_202010  | table | uxdb  | 2016 kB | 
 public | log_ins_202011  | table | uxdb  | 1984 kB | 
 public | log_ins_202012  | table | uxdb  | 1984 kB | 
 public | log_ins_history | table | uxdb  | 2016 kB | 
(14 rows)
```

父表无数据，各子表一切正常，传统分区表情况介绍到这儿。



#### 添加分区

添加分区属于分区表维护的常规操作之一，比如历史表范围分区到期之前需要扩分区，log_ins表为日志表，每个分区存储当月数据，假如分区'快到期了，可通过以下SQL扩分区，首先创建子表

```
CREATE TABLE log_ins_202101(CHECK(create_time>='2021-01-01' and create_time 
<'2021-02-01')) INHERITS (log_ins);
```

通常会多定义一些分区，这个操作要根据具体场景来进行。
之后创建相关索引

```
CREATE INDEX idx_log_ins_20210l_ctime ON log_ins_202101 USING btree(create_time); 
```

然后刷新触发器函数log_ins_ i nsert_ trigger（），添加相应代码，将符合路由规则的数据
插入新分区，详见之前定义的这个函数，这步完成后，添加分区操作完成，可通过\d+ log_ins命令查看log_ins 的所有分区。
这种方法比较直接，创建分区时就将分区继承到父表，如果中间步骤有错可能对生产
系统带来影响，比较推荐的做法是将以上操作分解成以下几个步骤，降低对生产系统的影
响
一创建分区

```
create table log_ins_202102(like log_ins including all); 
```

一添加约束

```
alter table log_ins_202102 add constraint log_ins_202102_create_time_check 
check(create_time >='2021-02-01' and create_time<'2021-03-01');
```

一刷新触发器函数log_ins_insert_trigger()

```
	ELSIF (NEW.create_time >='2021-01-01' and NEW.create_time<='2021-02-01') THEN 
		INSERT INTO log_ins_202101 VALUES (NEW.*); 	
```

函数刷新前建议先备份函数代码。

－－所有步骤完成后，将新分区log_ins_202102继承到父表log_ins 

```
ALTER TABLE log_ins_202102 INHERIT log_ins;
```

以上方法是将新分区所有操作完成后，再将分区继承到父表，降低了生产系统添加分区操作的风险，当然，在生产系统添加分区前建议在测试环境事先演练一把。

查看一下

```sql
\d+ log_ins
                                                              Table "public.log_ins"
   Column    |              Type              | Collation | Nullable |               Default               | Storage | Stats
 target | Description 
-------------+--------------------------------+-----------+----------+-------------------------------------
 id          | integer                        |           | not null | nextval('log_ins_id_seq'::regclass) | plain   |      
        | 
 user_id     | integer                        |           |          |                                     | plain   |      
        | 
 create_time | timestamp(0) without time zone |           |          |                                     | plain   |      
        | 
Triggers:
    insert_log_ins_trigger BEFORE INSERT ON log_ins FOR EACH ROW EXECUTE PROCEDURE log_ins_insert_trigger()
Child tables: log_ins_202001,
              log_ins_202002,
              log_ins_202003,
              log_ins_202004,
              log_ins_202005,
              log_ins_202006,
              log_ins_202007,
              log_ins_202008,
              log_ins_202009,
              log_ins_202010,
              log_ins_202011,
              log_ins_202012,
              log_ins_202101,
              log_ins_202102,
              log_ins_history
```

#### 删除分区

分区表的一个重要优势是对于大表的管理上十分方便，例如需要删除历史数据时可以直接删除一个分区，这比DELETE方式效率高了多个数量级，传统分区表删除分区通常有两种方法，第一种方法是直接删除分区

```sql
DROP TABLE log ins_202102 
```

就像删除普通表一样删除分区即可，当然删除分区前需再三确认是否需要备份数据；
另一种比较推荐的删除分区方法是先将分区的继承关系去掉

```sql
ALTER TABLE log ins 202102 NO INHERIT log ins; 
```

执行以上命令后，log_ins_ 202102分区不再属于分区表log_ins的分区，但log_ins_202102表依然保留可供查询，这种方式相比方法一提供了一个缓冲时间，属于比较稳妥的删除分区方法，因为在拿掉子表继承关系后，只要没删除这个子表，还可以使子表重新继承父表。

#### 注意事项

1. 当往父表上插入数据时，需事先在父表上创建路由函数和触发器，数据才会根据分区键路由规则插入到对应分区中，目前仅支持范围分区和列表分区。

2. 分区表上的索引、约束需要使用单独的命令创建，目前没有办法一次性自动在所有分区上创建索引、约束。

3. 父表和子表允许单独定义主键，因此父表和子表可能存在重复的主键记录，目前不支持在分区表上定义全局主键。

4. UPDATE时不建议更新分区键数据，特别是会使数据从一个分区移动到另一分区的场景，可通过更新触发器实现，但会带来管理上的成本。

5. 性能方面：根据本节的测试数据和测试场景，传统分区表根据非分区键查询相比普通表性能差距较大，因为这种场景下分区表会扫描所有分区；根据分区键查询相比普通表性能有小幅降低，而查询分区表子表性能相比普通表略有提升；使用分区表不一定能提升性能，如果业务模型90%（估算的百分比，意思是大部分）以上的操作都能基于分区健操作，并且SQL可以定位到子表，这时建议使用分区表。

   

### 内置分区表

一个重量级新特性是支持内置分区表，用户不需要预先在父表上定义INSERT、 DELETE、 UPDATE触发器，对父表的DML操作会自动路由到相应分区，相比传统分区表大幅度降低了维护成本，目前仅支持范围分区和列表分区

#### 创建分区表

创建分区表的主要语法包含两部分：创建主表和创建分区。
创建主表时须指定分区方式，可选的分区方式为RANGE范围分区或LIST列表分区，并指定宇段或表达式作为分区键。
创建分区时必须指定是哪张表的分区，同时指定分区策略partition_bound_ spec，如果是范围分区，partition_bound_spec须指定每个分区分区键的取值范围，如果是列表分区partition_ bound_ spec，需指定每个分区的分区键值。

##### 1 创建父表，指定分区键和分区策略。

```
CREATE TABLE log_par( 
id serial , 
user_id int4 , 
create_time timestamp(0) without time zone 
) PARTITION BY RANGE(create_time); 
```

##### 2 创建分区，创建分区时须指定分区表的父表和分区键的取值范围，注意分区键的范围不要有重叠，否则会报错。

表log_par指定了分区策略为范围分区，分区键为create_time宇段。
创建分区，并设置分区的分区键取值范围

```sql
create table log_par_his partition of log_par for values from (MINVALUE) to ('2020-01-01'); 
create table log_par_202001 partition of log_par for values from ('2020-01-01') to (' 2020-02-01'); 
create table log_par_202002 partition of log_par for values from ('2020-02-01') to (' 2020-03-01');
create table log_par_202003 partition of log_par for values from ('2020-03-01') to (' 2020-04-01');
create table log_par_202004 partition of log_par for values from ('2020-04-01') to (' 2020-05-01');
create table log_par_202005 partition of log_par for values from ('2020-05-01') to (' 2020-06-01');
create table log_par_202006 partition of log_par for values from ('2020-06-01') to (' 2020-07-01');
create table log_par_202007 partition of log_par for values from ('2020-07-01') to (' 2020-08-01');
create table log_par_202008 partition of log_par for values from ('2020-08-01') to (' 2020-09-01');
create table log_par_202009 partition of log_par for values from ('2020-09-01') to (' 2020-10-01');
create table log_par_202010 partition of log_par for values from ('2020-10-01') to (' 2020-11-01');
create table log_par_202011 partition of log_par for values from ('2020-11-01') to (' 2020-12-01');
create table log_par_202012 partition of log_par for values from ('2020-12-01') to (' 2021-01-01');

/*MINVALUE原为UNBOUNDED，含义是没有边界，该常量在本版中无效*/
```

##### 3 在分区上创建相应索引

通常情况下分区键上的索引是必须的，非分区键的索引可根据实际应用场景选择是否创建。

```sql
CREATE INDEX idx_log_par_his_ctime ON log_par_his USING btree(create_time);
CREATE INDEX idx_log_par_202001_ctime ON log_par_202001 USING btree(create_time); 
CREATE INDEX idx_log_par_202002_ctime ON log_par_202002 USING btree(create_time);
CREATE INDEX idx_log_par_202003_ctime ON log_par_202003 USING btree(create_time);
CREATE INDEX idx_log_par_202004_ctime ON log_par_202004 USING btree(create_time);
CREATE INDEX idx_log_par_202005_ctime ON log_par_202005 USING btree(create_time);
CREATE INDEX idx_log_par_202006_ctime ON log_par_202006 USING btree(create_time);
CREATE INDEX idx_log_par_202007_ctime ON log_par_202007 USING btree(create_time);
CREATE INDEX idx_log_par_202008_ctime ON log_par_202008 USING btree(create_time);
CREATE INDEX idx_log_par_202009_ctime ON log_par_202009 USING btree(create_time);
CREATE INDEX idx_log_par_202010_ctime ON log_par_202010 USING btree(create_time);
CREATE INDEX idx_log_par_202011_ctime ON log_par_202011 USING btree(create_time);
CREATE INDEX idx_log_par_202012_ctime ON log_par_202012 USING btree(create_time);
```

##### 向分区表插入数据

```sql
INSERT INTO log_par(user_id,create_time) 
SELECT round(100000000*random()) , generate_series('2019-12-01'::date,'2020-12-31'::date,'1 minute'); 
```

#### 查看分区

```sql
\dt+ log_par*
                        List of relations
 Schema |      Name      | Type  | Owner |  Size   | Description 
--------+----------------+-------+-------+---------+-------------
 public | log_par        | table | uxdb  | 0 bytes | 
 public | log_par_202001 | table | uxdb  | 2016 kB | 
 public | log_par_202002 | table | uxdb  | 1920 kB | 
 public | log_par_202003 | table | uxdb  | 2016 kB | 
 public | log_par_202004 | table | uxdb  | 1984 kB | 
 public | log_par_202005 | table | uxdb  | 2016 kB | 
 public | log_par_202006 | table | uxdb  | 1984 kB | 
 public | log_par_202007 | table | uxdb  | 2016 kB | 
 public | log_par_202008 | table | uxdb  | 2016 kB | 
 public | log_par_202009 | table | uxdb  | 1984 kB | 
 public | log_par_202010 | table | uxdb  | 2016 kB | 
 public | log_par_202011 | table | uxdb  | 1984 kB | 
 public | log_par_202012 | table | uxdb  | 1984 kB | 
 public | log_par_his    | table | uxdb  | 2016 kB | 
```

#### 添加分区

给log_par增加一个分区

```sql
create table log_par_202101 partition of log_par for values from ('2021-01-01')to ('2021-02-01'); 
```

给分区创建索引

```sql
CREATE INDEX idx_log_par_20210_ctime ON log_par_202101 USING btree(create_time) ; 
```

#### 删除分区

DROP分区方式删除

```sql
DROP TABLE log_par_202101; 
```

DROP方式直接将分区和分区数据删除，删除前需确认分区数据是否需要备份，避免数据丢失；

解绑分区

```sql
ALTER TABLE log_par DETACH PARTITION log_par_202101; 
```

解绑分区只是将分区和父表间的关系断开，分区和分区数据依然保留，这种方式比较稳妥，如果后续需要恢复这个分区，通过连接分区方式恢复分区即可

```sql
ALTER TABLE log_par ATTACH PARTITION log_par_202101 FOR VALUES FROM ('2021-01-01') TO ('2021-02-01');
```

连接分区时需要指定分区上的约束。

#### 性能比较

- 内置分区表根据非分区键查询相比普通表性能差距较大，因为这种场景分区表的执行计划会扫描所有分区；
- 内置分区表根据分区键查询相比普通表性能有小幅降低，而查询分区表子表性能比普通表略有提升；
- 以上两个场景除了场景二直接检索分区表子表，性能相比普通表略有提升，其他测试项分区表比普通表性能都低，
- 出于性能考虑对生产环境业务表做分区表时需慎重，使用分区表不一定能提升性能，
- 内置分区表相比传统分区表省去了创建触发器路由函数、触发器操作，减少了维护成本，有较大管理方面的优势。
# UXDB实例

## 表膨胀处理一例

VACUUM  -- 清理与分析数据库工具

### 1、建表、模拟数据

```
create table tab_vac(id int8,name varchar);
CREATE TABLE
insert into tab_vac values(1,'uxdb');
INSERT 0 1
insert into tab_vac values(2,'sql');
INSERT 0 1
insert into tab_vac values(3,'china');
INSERT 0 1
select lp as linetuple,t_xmin as 插入元组的txid,t_xmax as 删除或者锁定的txid,t_field3 as 插入命令id,t_ctid as 指向元组自身的id from heap_page_items(get_raw_page('tab_vac',0));

 linetuple | 插入元组的txid | 删除或者锁定的txid | 插入命令id | 指向元组自身的id 
-----------+----------------+--------------------+------------+------------------
         1 |       61813668 |                  0 |          0 | (0,1)
         2 |       61813669 |                  0 |          0 | (0,2)
         3 |       61813670 |                  0 |          0 | (0,3)
(3 rows)
```

### 2 删除一条数据

```
delete from tab_vac where id=2;
DELETE 1
select lp as linetuple,t_xmin as 插入元组的txid,t_xmax as 删除或者锁定的txid,t_field3 as 插入命令id,t_ctid as 指向元组自身的id from heap_page_items(get_raw_page('tab_vac',0));
 linetuple | 插入元组的txid | 删除或者锁定的txid | 插入命令id | 指向元组自身的id 
-----------+----------------+--------------------+------------+------------------
         1 |       61813668 |                  0 |          0 | (0,1)
         2 |       61813669 |           61813671 |          0 | (0,2)
         3 |       61813670 |                  0 |          0 | (0,3)
(3 rows)
```

#### 3 清理表

```
vacuum full tab_vac;
VACUUM
select lp as linetuple,t_xmin as 插入元组的txid,t_xmax as 删除或者锁定的txid,t_field3 as 插入命令id,t_ctid as 指向元组自身的id from heap_page_items(get_raw_page('tab_vac',0));

 linetuple | 插入元组的txid | 删除或者锁定的txid | 插入命令id | 指向元组自身的id 
-----------+----------------+--------------------+------------+------------------
         1 |       61813668 |                  0 |          0 | (0,1)
         2 |       61813670 |                  0 |          0 | (0,2)
(2 rows)
```

#### 4 其它

如果您希望自动清理，请在uxsinodb.conf中修改，去掉#autovacuum = on  的“#”，注意其以下参数设置。

```
#------------------------------------------------------------------------------
AUTOVACUUM PARAMETERS
#------------------------------------------------------------------------------
autovacuum = on                        # Enable autovacuum subprocess?  'on'					 		  
```

如果您有分区表，您正在手动使用VACUUM或 ANALYZE命令，别忘了分别在每个分区上运行它们。
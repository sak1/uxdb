# UXDB实例

## Returning函数返回DML操作数据

UXDB 的Returning特性可以返回DML修改的数据，可以返回 INSERT、UPDATE、DELETE语句时返回对应操作的数据。优点在于不需要额外的SQL就可以获取操作值，方便程序开发，示例演示一下：

操作方法为在DML语句后加" returning *"，或"returning 列名".

### 1 返回插入的数据

INSERT语句后接RETURNING属性返回插入值，先创建测试表test_return，再插入并返回插入的整行数据。

```sql
/*建表*/
create table test_return(id serial,flag char(10));
CREATE  TABLE 

/*插入并返回值，returning＊ ：返回表插入的所有字段值，也可以返回指定字段值*/
insert into test_return(flag) values('王维') returning *; 
 id | flag 
----+------
  1 | 王维
(1 row)
INSERT 0 1

/*插入并返回值，returning flag：返回flag字段插入值*/
insert into test_return(flag) values('王国维') returning flag; 
 flag 
---------
王国维
(1 row)
INSERT 0 1
```



```sql

```

### 2 返回更新后数据

UPDATE后接RETURNING属性，返回UPDATE语句更新后的值，例如：

```sql
update test_return set flag ='杜甫' where id=1 returning *; 
 id | flag 
----+------
  1 | 杜甫
(1 row)

UPDATE 1
```

### 3 返回删除的数据

DELETE后接RETURNING属性，返回删除的数据，如下所示：

```sql
delete from test_return where id=2 returning *; 
 id | flag 
----+------
  2 | 王国维
(1 row)

DELETE 1
```



author:steven tong

date:2021/3
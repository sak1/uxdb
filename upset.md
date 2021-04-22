# UXDB实例

## UPSERT在数据冲突中的应用

日志数据应用场景中，通常会在事务中批量插入日志数据，如果其中有一条数据违反表中的约束，整个插入事务就会回滚，UXDB的UPSERT特性能解决这一问题。我们通过例子来理解UPSERT的功能，

首先定义一张用户登录日志表并插入一条数据

```sql
CREATE TABLE user_logins(user_name text primary key,login_cnt int4,last_login_time timestamp(0) without time zone);  
/*建表 user_logins*/
select * from user_logins 
/*看下结构 */
```

```sql
INSERT INTO user_logins(user_name, login_cnt) VALUES('steven',1);    
/*插入数据*/
select * from user_logins  
/*再看一下*/
```


user_logins表user_name字段定义了主键，这时，批量插入数据中如果有重复，会报错， 如下所示：

```sql
INSERT INTO user_logins(user_name,login_cnt) VALUES('mary',1),('steven',1);  
/*插入数据*/
DETAIL:  Key (user_name)=(steven) already exists.  
/*报错*/
```

上面的SQL试图插入两条数据，mary这条数据不违反主键冲突，steven这条数据违反主键冲突，结果两条数据都不能插入。下面我们使用UPSERT处理冲突的数据，如当插入的数据冲突时不报错，同时更新冲突的数据，如下所示：

```sql
INSERT INTO user_logins(user_name, login_cnt) VALUES('mary',1),('steven',1) 
ON CONFLICT(user_name) 
DO UPDATE SET
login_cnt=user_logins.login_cnt+EXCLUDED.login_cnt,last_login_time=now();
```

insert语句插入两条数据，并设置规则：当数据冲突时将登录次数字段login_cnt值加1， 同时更新最近登录时间last_login_time, ON  CONFLICT(user_name）定义冲突类型为user_name字段，DO UPDATE SET是指冲突动作，后面定义了一个UPDATE语句。

上述SET命令中引用了user_loins表和内置表EXCLUDED， 引用原表user_loins访问表中己存在的冲突记录， 内置表EXCLUDED引用试图插入的值，再次查询表user_login, 例如：

```sql
SELECT * FROM user_logins; 
user_name | login_cnt |   last_login_time   
-----------+-----------+---------------------
 mary      |         1 | 
 steven    |         2 | 2021-03-24 15:25:49
(2 rows)
```

一方面，我们看steven这条冲突数据更新了login_cnt和last_login_time字段，面mary这条记录己正常插入。

也可以定义数据冲突后什么都不干，这时需指定DO NOTHING属性，例如：

```sql
INSERT INTO user_logins(user_name,login_cnt) VALUES('jack',1),('steven',1) 
ON CONFLICT(user_name) DO NOTHING; 
```

再次查询表数据，新的数据jack这条己插入到表中，冲突的数据steven这行没变，结果如下：

```sql
SELECT * FROM  user_logins;
user_name | login_cnt |   last_login_time   
-----------+-----------+---------------------
 mary      |         1 | 
 steven    |         2 | 2021-03-24 15:25:49
 jack      |         1 | 
```


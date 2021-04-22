# UXDB实例

## 审计功能

审计功能目前限于安全版本，我们择机另作介绍，以下浅尝辄止、仅供参考

数据库审计功能用于分析发现违规操作、异常行为、潜在侵害。即uxaudit插件,该插件用于创建审计员uxad、创建审计表uxaudit.session、uxaudit.logon、uxaudit.log_event等，在插件初始化过程中创建DDL、DML、FUNCTION审计钩子函数，对后续用户连接数据库、操作数据库行为进行审计。

审计用户uxad通过查看vw_audit_event审计视图分析发现违规操作、异常行为、潜在侵害等。

## 1 审计信息表

#### log_event表

session_id	登陆的session的ID
session_line_num	在该session中执行的SQL的代号
log_time	日志时间
command	执行的SQL命令
error_severity	日志级别
sql_state_code	SQL执行完毕的状态码
virtual_transaction_id	虚拟事务ID
transaction_id	事务ID
message	描述
detail	详情
hint	该SQL执行的提示信息
query	查询
query_pos	查询的position
internal_query	内部查询
internal_query_pos	内部查询position
context	上下文
location	Location信息

#### session 表

字段名	含义
session_id	登陆的session的ID
process_id	进程ID
session_start_time	session开始时间
user_name	登陆用户名
application_name	进程名字
connection_from	从何处进行的连接

#### logon 表

字段名	含义
user_name	登陆用户名
last_success	上一次成功登录时间
current_success	当前成功登陆的时间
last_failure	最后一次失败的时间
failures_since_last_success	从最后一次成功登陆时间之后，连续登陆失败的次数

#### audit_statement 表

字段名	含义
session_id	登陆的sessionID
statement_id	SQL statement ID
state	执行状态
error_session_line_num	错误的statement ID在当前session中是第几个SQL命令

#### audit_substatement 表

字段名	含义
session_id	登陆的sessionID
statement_id	SQL statement ID
substatement_id	子SQL statement ID
substatement	SQL正文
parameter	SQL参数列表

#### audit_substatement_detail 表

session_id	登陆的sessionID
statement_id	SQL statement ID
substatement_id	子SQL statement ID
session_line_num	在该session中执行的SQL的代号
audit_type	审计类型 session - 主体操作 或者object - 客体操作
class	分类
command	命令
object_type	客体类型
object_name	客体名称

#### audit_time表

last_row_time	最后计入时间

例如：通过sql来查询审计
uxop查询用户登录情况



### 2 自定义审计

设置审计开关来进行对应的审计。

uxaudit.log审计类型。选项有：
none:所有都不审计
all:所有都审计。
read:审计读，如查询。
write:审计写，如插入，更新，删除，清除和copy。
function:函数审计，如do。
role: 与角色和权限相关的审计，如授予、收回、创建、更改、删除角色。
ddl：所有ddl的审计。
misc：杂项命令审计，例如DISCARD、FETCH、CHECKPOINT、VACUUM。
默认值是read,write,ddl,role,function,misc

- uxaudit.log_catalog 系统表查询审计开关。缺省为开。
- uxaudit.log_level 审计日志级别设置。缺省为LOG级别。还可以设置FATAL，PANIC。
- uxaudit.log_parameter 该开关确定是否要记录SQL执行命令时候的参数。缺省为打开
- uxaudit.log_relation 是否在对session级别进行审计时，对每个对relation（TABLE，VIEW等）操作的SELECT或者其他DML创建单独的log记录。缺省为关闭。
- uxaudit.log_statement_once 是否只在第一次出现SQL statement的时候进行审计。缺省为关闭。
- uxaudit.role 审计员，缺省为audit。

审计设定可以针对主体（Session）或者客体（object），具体示例如下：
审计策略由审计员通过alter...set设置，并且重启集群生效。

#### 设置审计策略为写，并重启集群使之生效。

```sql
alter system set uxaudit.log = write;
```

#### 创建测试用户

```sql
create user audit_test password '1qaz!QAZD';
```

#### 使用上面创建的用户创建测试表并分别进行对测试表进行读写

```sql
create table industry(id int,name text);
insert into industry (id,name)values (11,'polo');
select * from industry;
```

#### 查看审计日志

```sql
select * from uxaudit.vw_audit_event where user_name = 'audit_test';
```

审计日志只记录了写（insert）的日志，对于其他（create和select）没有进行记录。

### 3 日志自动覆盖

为了防止日志存储过大，导致磁盘占满。UXDB可以设置日志自动覆盖，并在最新的日志中给出相应的提示。 
通过修改uxsinodb.conf中的参数来实现，需要重启集群

```shell
logging_collector = on
log_directory = 'ux_log'
log_filename = 'uxsinodb-%a.log' 
log_file_mode = 0600 
log_truncate_on_rotation = on
log_rotation_size = 100MB  #当达到100M的时候自动覆盖
```

 可使用以下方法来产生大量日志。

#### 创建一次插入100W数据的存储过程

```sql
create or replace function insert_test() returns   
boolean as  
$body$  
declare ii integer;  
begin  
	ii:=1;  
	for ii in 1..1000000 loop  
		insert into testspace(id1, id2) values (116, ii);  
	end loop;  
	return true;  
end;  
$body$  
language pluxsql;
```

#### 多次执行上述存储过程插入数据，使数据大于100M

```
select insert_test();
```


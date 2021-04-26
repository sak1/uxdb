# UXDB实例

## 审计功能

数据库审计功能可用于分析发现违规操作、异常行为、潜在侵害。

#### 安装uxaudit插件

修改配置

```
$ vi /home/uxdb/uxdbinstall/dbsql/data/uxsinodb.conf
shared_preload_libraries = 'uxaudit,ux_stat_statements'
```

重启实例

```
./ux_ctl restart -D ../data
```

安装扩展

```
create extension uxaudit;
```

查看设置

```
select name,setting from ux_settings where name like 'uxaudit%';
            name            | setting 
----------------------------+---------
 uxaudit.log                | none
 uxaudit.log_catalog        | on
 uxaudit.log_client         | off
 uxaudit.log_level          | log
 uxaudit.log_parameter      | off
 uxaudit.log_relation       | off
 uxaudit.log_statement_once | off
 uxaudit.role               | 
(8 rows)
```

修改配置

```
vi /home/uxdb/uxdbinstall/dbsql/data/uxsinodb.conf

uxaudit.log = 'all, -misc'  # 除杂项命令外的都审计

#none:所有都不审计
#all:所有都审计
#read:审计读，如查询。
#write:审计写，如插入，更新，删除，清除和copy。
#function:函数审计，如do。
#role: 与角色和权限相关的审计，如授予、收回、创建、更改、删除角色。
#ddl：所有ddl的审计。
#misc：杂项命令审计，例如DISCARD、FETCH、CHECKPOINT、VACUUM。  默认值是read,write,ddl,role,function,misc
  
uxaudit.log_catalog = on	#系统表查询审计开关。缺省为开。
uxaudit.log_client = on 	#指定日志消息对于诸如uxsql之类的客户端进程是否可见,默认为关闭。
uxaudit.log_level = log	 	#审计日志级别设置。缺省为LOG级别。还可以设置FATAL，PANIC。
uxaudit.log_parameter = on	#该开关确定是否要记录SQL执行命令时候的参数。缺省为打开
uxaudit.log_relation = on	#是否在对会话级别进行审计时，对每个对relation（TABLE，VIEW等）操作的SELECT或者其他DML创建单独的log记录，缺省为关闭。
uxaudit.log_statement_once = on	#是否只在第一次出现SQL statement的时候进行审计。缺省为关闭。

#uxaudit.role = 	#审计员，缺省为audit。
```

重载配置

```
./uxsql
SELECT ux_reload_conf();
```

### 查看审计日志

库表操作

```
set uxaudit.log = 'all, -misc';
create table account(id int,name text,password text,description text);
insert into account (id, name, password, description) values (1, 'user1', 'HASH1', 'blah, blah');
select * from account;
```

```shell
tailf ../data/log/uxsinodb-2021-04-26_180943.log 
```

```
2021-04-26 19:03:21.956 CST [6032] LOG:  AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.account,"create table account(id int,name text,password text,description text);",<none>
2021-04-26 19:03:26.797 CST [6032] LOG:  AUDIT: SESSION,2,1,WRITE,INSERT,TABLE,public.account,"insert into account (id, name, password, description) values (1, 'user1', 'HASH1', 'blah, blah');",<none>
2021-04-26 19:03:30.944 CST [6032] LOG:  AUDIT: SESSION,3,1,READ,SELECT,TABLE,public.account,select * from account;,<none>
```

### 日志自动覆盖

为了防止日志存储过大，导致磁盘占满。您可以通过修改uxsinodb.conf中的参数来实现，需要重启数据库

```shell
logging_collector = on
log_directory = 'ux_log'
log_filename = 'uxsinodb-%a.log' 
log_file_mode = 0600 
log_truncate_on_rotation = on
log_rotation_size = 100MB  #当达到100M的时候自动覆盖
```

重启实例

```
./ux_ctl restart -D ../data
```


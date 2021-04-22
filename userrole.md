# UXDB实例

## 用户与权限详解

### 用户概述

通用版数据库有两类用户：超级用户和普通用户；
安全版数据库超级用户又被三类，分别是系统管理员、安全保密管理员、审计管理员。
超级管理员不做权限检查；

数据库中的用户和角色不做区分，唯一的区别是login权限。
和Oracle12c 以上版本数据库不同，UXDB没有通用用户和本地用户的区别，用户建立在实例级别，通过权限对数据库对象进行控制。

### 权限体系

UXDB的权限体系类似Linux操作系统，有用户和组概念，集群（实例）有数据库对象集合的概念，而数据库是多模式（schema）的集合，每一个模式下又包含多个数据库对象（表、视图、序列等）。UXDB权限可以划分为：实例权限->表空间权限->数据库权限->schema权限->object权限

> 集群 翻译成cluster，实际对就为实例，可以理解为初始化的数据目录，如：/home/uxdb/uxdbinstall/dbsql/data

都有哪些权限呢？

- 实例级权限：用户、角色、数据创建、登录以及复制；
- 表空间权限：创建对象；
- 数据库级权限：连接、创建方案等；
- 方案级权限：使用方案以及在方案中创建对象；
- 表级权限：增删改查等权限；
- 列级权限：允许或限制对列的访问。

以上权限绝可以用grant授予，用revoke进行回收。

UXDB分为系统权限和对象权限：

系统权限——针对数据库、用户、角色的权限，如连接数据库、创建数据库、创建用户等，权限包括LOGIN、PASSWORD、SUPERUSER、CREATEDB、CREATEROLE、INHERIT等。

对象权限——针对数据库对象，如表、序列、函数等客体上执行动作的权限，权限包括select、insert、update、delete、references、trigger、create、connect、temporary、execute和usage等。

角色属性信息在系统表ux_authid里。ux_roles是ux_authid公开的视图。角色之间的成员关系在ux_auth_members表里，可以通过select查阅。

### 特殊权限

UXDB中的超级管理员相当于Oracle的sys用户，拥有数据库的所有权限，访问权限限制可以实例层面设置；另一个特殊用户是public，准确说是角色public，因为uxdb中用户和角色是不做区分的，用户和角色唯一不同是login权限。public角色代表所有人，当把一项权限授予public时，相当于所有用户都具备这个权限。

在UXDB实例被初始化后，会默认默认建立一个名为uxdb的超级管理员。。

在UXDB实例被初始化之后，默认角色public拥有connect权限，可以连接到任意数据库；

在每一个数据库下，默认了一个public的schema，默认了public角色可以在public schema下创建object的权限，因此，UXDB在实例被初始化之后，默认的权限是较大的，在生产环境中需要做限制。例如：收回public角色的connect权限，如下：

```
revoke CONNECT on DATABASE uxdb from PUBLIC;
```

也可以收回public的所有权限，执行：

```
revoke ALL on DATABASE uxdb from PUBLIC ;
```

all在数据库上的权限代表有 CREATE | CONNECT | TEMPORARY | TEMP

这样，新建一个用户之后，要使新建用户能够访问某个数据库，就需要授予connect权限，否则，新建用户是连接不了任何数据库的。
例子1：
（1）创建用户user1

```
create user user1 password 'steven';
CREATE ROLE
```

（2）使用user1连接uxdb数据库

```
uxsql -U user1 -d uxdb
uxsql: FATAL:  permission denied for database "uxdb"
DETAIL:  User does not have CONNECT privilege.
```

收回在public 角色在public下的所有权限：

```
revoke ALL on SCHEMA public from PUBLIC ;
```

ALL代表CREATE | USAGE
收回后，用户具备连接权限，依然不能再数据库中创建任何object

例子2
（1）授予user1连接数据库uxdb的权限

```
grant CONNECT on DATABASE uxdb to user1 ;
```

（2）使用user1连接uxdb数据库

```
uxsql -U user1 -d uxdb

The license due date is: 2021-10-01 00:00:00
It's commercial license.
uxsql
Type "help" for help.
```



（2）新建表

```
create table public.t1(id int);
ERROR:  permission denied for schema public
LINE 1: create table public.t1(id int);
```


甚至连选择public  schema的权限都没有：

```sql
create table t1(id int);
ERROR:  no schema has been selected to create in
LINE 1: create table t1(id int);
```

使用管理员授予usage访问schema的权限：
grant USAGE on SCHEMA public to user1;

此时user1 有使用usage权限，但无创建权限：

```
create table t1(id int);
ERROR:  permission denied for schema public
LINE 1: create table t1(id int);
```

再次使用管理员授予create权限

```
grant CREATE on SCHEMA public to user1 ;
```

此时user1 就具有了创建对象权限：

```
create table t1(id int);
CREATE TABLE
```

有在schema上使用和创建object的权限，但不代表具备创建schema权限

```
create schema user1;
ERROR:  permission denied for database uxdb
```

如果要用于创建schema权限，需要在数据库层面授予create权限。

### 权限解读

如果一个object对象是某一用户创建，那么用户是object的拥有者，object拥有者拥有对该object的一切权限，包括：SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER 等权限。拥有者可以将该对象的权限授予其他用户。

总结，UXDB中权限含义如下：
rolename=xxxx -- privileges granted to a role  
                 =xxxx -- privileges granted to PUBLIC  

            r -- SELECT ("read")  
            w -- UPDATE ("write")  
            a -- INSERT ("append")  
            d -- DELETE  
            D -- TRUNCATE  
            x -- REFERENCES  
            t -- TRIGGER  
            X -- EXECUTE  
            U -- USAGE  
            C -- CREATE  
            c -- CONNECT  
            T -- TEMPORARY  
      arwdDxt -- ALL PRIVILEGES (for tables, varies for other objects)  
            * -- grant option for preceding privilege  
      
        /yyyy -- role that granted this privilege  

```
\z
                              Access privileges
 Schema |   Name   | Type  | Access privileges | Column privileges | Policies 
--------+----------+-------+-------------------+-------------------+----------
 public | capitals | table | uxdb=arwdDxt/uxdb |                   | 
 public | cities   | table | uxdb=arwdDxt/uxdb |                   | 
 public | t1       | table |                   |                   | 
(3 rows)
```

在此列表中，对于表capitals，uxdb用户有插入、查询、修改、删除、截断（truncate）、REFERENCES（允许创建外键）、建触发器权限，此权限是uxdb用户授予的；对于表t1,访问权限一列为空，表示只有拥有者有所有权限，其他用户（超级管理员除外）都没权限。
使用超级管理员收回t1的查询权限，如下：

```
revoke SELECT on TABLE t1 from user1;
REVOKE

\z
                               Access privileges
 Schema |   Name   | Type  | Access privileges  | Column privileges | Policies 
--------+----------+-------+--------------------+-------------------+----------
 public | capitals | table | uxdb=arwdDxt/uxdb  |                   | 
 public | cities   | table | uxdb=arwdDxt/uxdb  |                   | 
 public | t1       | table | user1=awdDxt/user1 |                   | 
(3 rows)

 \c - user1
You are now connected to database "uxdb" as user "user1".
select * from t1;
ERROR:  permission denied for relation t1
```



### 主机认证权限

在uxdb中，可以限制超级管理员登录数据库，需要修改主机认证文件ux_hba.conf，例如，这里限制超级管理员uxdb从本地登录，只允许user1 登录如下：

```
"local" is for Unix domain socket connections only

#local   all             all                                     trust
local   uxdb             user1                                     trust
不允许超级管理员登录也可以配置为：
local   uxdb             uxdb                                     reject
```

发送重新读取主机认证文件给实例

```
ux_ctl reload -D ~/localdb01/
```

或者使用信号量

```
kill -s SIGHUP 3161
```

3161表示主进程编号，可以通过ps -ef|grep uxdb查看

再次使用超级管理员登录会收到一下拒绝提示：

```
uxsql -U uxdb -d uxdb
uxsql: FATAL:  no ux_hba.conf entry for host "[local]", user "uxdb", database "uxdb", SSL off
```

使用user1登录：

```
uxsql -U user1 -d uxdb

The license due date is: 2021-10-01 00:00:00
It's commercial license.
uxsql 
Type "help" for help.
```

登录数据库方式也可以在ux_hba.conf中设置，以上是使用操作系统认证，不需要密码即可登录。修改为需要使用密码登录：

```
#local   all             all                                     trust
local   uxdb             user1                                     md5
```

再次发送信号给数据库主进程

```
ux_ctl reload -D ~/localdb01/

uxsql -U user1 -d uxdb
Password for user user1: 

The license due date is: 2021-10-01 00:00:00
It's commercial license.
uxsql (10.0)
Type "help" for help.
```

### 系统权限

系统的CREATEDB、CREATEROLE等权限，是在创建用户时或者修改用户时授予的，通过create user 、create role 、 alter  user和alter role授予，例如帮助：

\h create user
Command:     CREATE USER
Description: define a new database role
Syntax:
CREATE USER name [ [ WITH ] option [ ... ] ]

where option can be:

      SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | REPLICATION | NOREPLICATION
    | BYPASSRLS | NOBYPASSRLS
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED ] PASSWORD 'password'
    | VALID UNTIL 'timestamp'
    | IN ROLE role_name [, ...]
    | IN GROUP role_name [, ...]
    | ROLE role_name [, ...]
    | ADMIN role_name [, ...]
    | USER role_name [, ...]
    | SYSID uid


如果要收回用户的系统权限，可以通过alter user或者alter role，比如给user1授予创建用户权限，然后再收回：
alter user user1 createrole ;

收回权限则是：
alter user user1 nocreaterole ;

### 权限级联

在Oracle数据中，系统权限使用with admin option授予后，在权限收回时，不级联回收；对象权限使用 with grant option授予后，权限回收时是级联回收的。在UXDB中，系统权限没有级联的概念；对象权限可以通过with grant option授予,在进行权限回收时，必须显示指定级联回收，否则提示有权限依赖，举例如下：
（1）使用超级管理员uxdb创建一张表，将表的查询权限授予user1，同时允许user1将此权限授予其他用户。

```
create table t2(id int);
CREATE TABLE
```

```
grant SELECT on TABLE t2 to user1 with grant option ;
GRANT
```

（2）使用user1登录，user 将权限授予steven用户

```
\c - user1
You are now connected to database "uxdb" as user "user1".

grant  SELECT on t2 TO steven;
GRANT
```

（3）使用超级管理员回收user1对表t2的查询权限

```sql
\c - uxdb
You are now connected to database "uxdb" as user "uxdb".
revoke SELECT on TABLE t2 from user1;
ERROR:  dependent privileges exist
HINT:  Use CASCADE to revoke them too.
```

必须使用cascade，才能把权限全部收回

```sql
revoke SELECT on TABLE t2 from user1 cascade;
REVOKE
```

### 默认权限

通过alter default privileges 可以定义默认权限。部分语法如下：

例子：
默认把查询模式public 下所有表的权限赋予steven
alter default privileges in schema public grant select on tables to steven;



```
\h alter default privileges 

Command:     ALTER DEFAULT PRIVILEGES
Description: define default access privileges
Syntax:
ALTER DEFAULT PRIVILEGES
    [ FOR { ROLE | USER } target_role [, ...] ]
    [ IN SCHEMA schema_name [, ...] ]
    abbreviated_grant_or_revoke

where abbreviated_grant_or_revoke is one of:

GRANT { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON TABLES
    TO { [ GROUP ] role_name | PUBLIC } [, ...] [ WITH GRANT OPTION ]

REVOKE [ GRANT OPTION FOR ]
    { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON TABLES
    FROM { [ GROUP ] role_name | PUBLIC } [, ...]
    [ CASCADE | RESTRICT ]
```

例子：
默认把查询模式public 下所有表的权限赋予steven

```sql 
alter default privileges in schema public grant select on tables to steven;
```
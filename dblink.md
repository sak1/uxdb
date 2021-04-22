# UXDB实例

## UXDB跨库查询

dblink 是UXDB 的一个模块，支持从数据库会话中连接到其他数据库。

### 安装 dblink

```
alert user uxdb with superuser;
\c template1
```

CREATE EXTENSION dblink; 

select * from ux_available_extensions;

\c uxdb   *abc为当前表

CREATE EXTENSION dblink; 

登录UXDB数据库，在uxdb=#下

```sql
CREATE EXTENSION dblink; 
```

### 本地跨库访问

首先在 uxdb 中建立 userinfo 表，随后在本地新建一个 localdb 数据库，并在其中建立一个 local_test 数据表。

```sql
*建表
CREATE TABLE userinfo (id int primary key, name text);    
*插值
INSERT INTO userinfo VALUES (1, 'Eric'), (2, 'Tom');
*查询一下
SELECT * FROM userinfo;
*建localdb库
CREATE DATABASE localdb;

*连接库
\c localdb
*建表
CREATE TABLE local_test (id serial primary key, ival int default 0, create_time timestamptz not null default now());
*插值
INSERT INTO local_test(ival) VALUES (1), (2), (3), (4);
*查询一下
SELECT * FROM local_test;
```

4条记录，OK,我们继续

在 uxdb 数据库中查询 local_test 表，就需要使用到 dblink 来访问了。首先，我们通过 dblink_connect 创建一个连接。

```sql
SELECT dblink_connect('local_dblink_test','dbname=localdb hostaddr=127.0.0.1 port=5432 user=uxdb password=mypassword');
```

再通过 dblink 执行查询。

```sql
SELECT * FROM dblink('local_dblink_test', 'SELECT * FROM local_test;') AS lt(id int, ival int, create_time timestamptz);
```

再把返回结果与本库中的表进行联合查询。

```sql
SELECT u.id, name, create_time FROM userinfo u JOIN dblink('local_dblink_test', 'SELECT * FROM local_test;') AS lt(id int, ival int, create_time timestamptz) on u.id = lt.id;
```

可以为dblink创建一个视图。

```sql
CREATE VIEW v_localdb_test AS SELECT * FROM dblink('local_dblink_test', 'SELECT * FROM local_test;') AS lt(id int, ival int, create_time timestamptz);

SELECT * FROM v_localdb_test;

SELECT u.id, name, create_time FROM userinfo u JOIN v_localdb_test v ON u.id = v.id;
```

> 备注： 在 local_test 表中 id 字段类型为 serial，但是在通过 dblink 查询时返回的结果类型不能使用 serial 类型。

### 远端跨库访问

远端跨库访问本质也是类似的，在配置 dblink_connect 连接参数时指明远端数据库的地址、端口、用户名和密码等信息。如：

```sql
SELECT dblink_connect('remote_dblink_test', 'dbname=remotedb hostaddr=192.168.10.24 port=5432 user=uxdb password=mypassword');
```

操作参考上面的<本地跨库访问>

### 关闭 dblink

当不需要在使用 dblink 访问外部数据库时，我们需要使用 dblink_disconnect 来关闭连接。首先，我们通过 dblink_get_connections 来查看现有的 dblink 连接，随后将其关闭。

```sql
SELECT dblink_get_connections();
SELECT dblink_disconnect('remote_dblink_test');
SELECT dblink_disconnect('local_dblink_test');
```

### 




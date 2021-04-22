# UXDB实例

## 如何锁定数据库实例为只读

数据库迁移割接时，首先要把主库切换到只读（锁定）状态，确保新的事务不会写入，导致数据不一致的情况。UXDB提供了 2 种只读锁定的方法：

1. 硬锁定：把数据库切换到恢复模式，不允许写操作，适合长时间禁止写入场景；
2. 软锁定：设置会话参数，把后续登录的会话、当前事务或用户设置为只读模式，允许随时恢复，适合临时禁止写入操作；

### 硬锁定

修改uxsinodb.conf

```
/*增加两行*/
recovery_target_timeline = 'latest'
standby_mode = on
```

如果是2.1.0.4及以前版本，请增加一个

```
cp /home/uxdb/uxdbinstall/dbsql/share/recovery.conf.sample /home/uxdb/uxdbinstall/dbsql/data/recovery.conf
```

重启数据库

```
select ux_is_in_recovery();  
 ux_is_in_recovery   
 t  
(1 row)  
```

```
create table t1(id int,info char(10));
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

无法建表吧，找个已有的表，\d 有个userionfo表

```
insert into userinfo values(3,'steven');
ERROR:  cannot execute INSERT in a read-only transaction
```

无法插入吧，哈哈！

### 硬解锁

```
cd /home/uxdb/uxdbinstall/dbsql/data/
mv recovery.conf recovery.done
```

重启、登录数据库。

```
./ux_ctl restart -D ../data
./uxsql
```

建个表试试

```
create table t1(id int,info char(10));
CREATE TABLE
```

再回到从前，一切还可以重演！

### 软锁定

锁定系统：设置系统级别的只读模式，数据库不需要重启也永久生效。

```
alter system set default_transaction_read_only=on;  
select ux_reload_conf();
```

试一下

```
insert into t1 values (1);  
ERROR:  cannot execute INSERT in a read-only transaction 
```

不可以操作了吧？哈哈

锁定会话：设置 Session 级别的只读模式，退出 SQL 交互窗口后失效。

```
set session default_transaction_read_only=on;
select ux_reload_conf();
```

锁定用户：设置指定登陆数据库的用户为只读模式，数据库不需要重启也永久生效。

```
alter user uxdb set default_transaction_read_only=on;
select ux_reload_conf();
```

### 软解锁

设置 default_transaction_read_only。

```sql
alter system set default_transaction_read_only=off;  
set session default_transaction_read_only=off;
alter user uxdb set default_transaction_read_only=off;
select ux_reload_conf();
```

```sql
insert into t1 values (1);  
INSERT 0 1
```

是不是很有趣？还有什么场景会用到呢？想想看哈。
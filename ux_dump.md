# UXDB实例

## SQL转储

SQL转储是创建一个由SQL命令组成的文件，重建与转储时状态一样的数据库。恢复时执行这些命令，从而恢复数据。

UXDB提供了工具ux_dump。基本用法是：

```
ux_dump dbname > dumpfile
```



### SQL转储备份：

1 先建个目录存储用于备份文件

```
mkdir /home/uxdb/bak
```

2 在/home/uxdb/uxdbinstall/dbsql/bin下执行

```
ux_dump uxdb > /home/uxdb/bak/uxdb
```

3 查看

```
ll /home/uxdb/bak
total 140648
-rw-rw-r--. 1 uxdb uxdb 144021225 Mar 19 10:58 uxdb
```

成功

### SQL转储恢复

基本用法：uxsql dbname < dumpfile

1 登录数据库，删除uxdb库

```
$./uxsql
#\c abc  *先连接其它库
```

```sql
删除uxdb库是危险，小心操作
#drop database uxdb; 
#create database uxdb; 
#\q
```

2 恢复

```
$uxsql uxdb < /home/uxdb/bak/uxdb

CREATE TABLE
CREATE VIEW
ALTER TABLE
COPY 911201

......
```

成功

### SQL转储全库数据备份

ux_dump每次只转储一个数据库，而且不会转储角色或表空间）的信息。为了支持便捷地转储一个数据库的全部内容。ux_dumpall备份给定集群中的每个
表空间定义。保留了集群内的所有数据。基本用法是：
ux_dumpall > dumpfile

如：

```shell
./ux_dumpall > /home/uxdb/bak/uxdb0304   
*uxdb0304为自定义的备份文件名
```

查看一下

```
uxdb=>ll /home/uxdb/bak
-rw-rw-r--. 1 uxdb uxdb 144029433 Mar 19 11:33 uxdb
```

> 注意：需要执行 1 登录数据库，删除uxdb库，参考上面的操作

### SQL转储全库数据恢复

可以使用uxsql恢复：uxsql -f dumpfile dbname

```
./uxsql -f /home/uxdb/bak/data0304 uxdb
```

成功！




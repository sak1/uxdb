# UXDB实例

## 安装mysql_fdw

设置环境变量（根据实际安装路径变更）

```bash
export PATH=/home/uxdb/uxdbinstall/dbsql/bin/:$PATH
export PATH=/usr/local/mysql/bin/:$PATH
make USE_PGXS=1    #编译 
make USE_PGXS=1 install  "  	#安装 
```

### 创建EXTENSION

```bash
CREATE EXTENSION mysql_fdw;
```

### 创建SERVER

```
 CREATE SERVER mysql_server
               FOREIGN DATA WRAPPER mysql_fdw
               OPTIONS (host '127.0.0.1', port '3306');
```

### 创建USER MAPPING

```sql
CREATE USER MAPPING FOR uxdb
              SERVER mysql_server
              OPTIONS (username 'root', password '123456');"
```

### 创建FOREIGN TABLE

```sql
CREATE FOREIGN TABLE warehouse(
              warehouse_id int,
              warehouse_name text,
              warehouse_created timestamp)
              SERVER mysql_server
              OPTIONS (dbname 'mysql', table_name 'warehouse');"
```

### 创建TABLE

```sql
 CREATE TABLE warehouse(
              warehouse_id int primary key not null,
              warehouse_name text,
              warehouse_created timestamp
              );"
```

### 插入数据

```
INSERT INTO warehouse values (1, 'UPS', now());
INSERT INTO warehouse values (2, 'TV', now());
INSERT INTO warehouse values (3, 'Table', now());
```

输入 select * from warehouse;
进入uxsinoSQL控制台输入 
              select * from warehouse;"

### 修改数据

```
update warehouse set warehouse_name='new name' where warehouse_id=2;
select * from warehouse;
```

进入uxsinoSQL控制台输入 
              select * from warehouse;"

### 删除数据

```
delete from warehouse where warehouse_id=3;
select * from warehouse;
```


进入uxsinoSQL控制台输入 
              select * from warehouse;"

### 插入数据

```
INSERT INTO warehouse values (3, 'Table', now());
INSERT INTO warehouse values (4, 'NEWS', '2016-06-02 10:00:00');
select * from warehouse;
```

进入mysql控制台输入 
              select * from warehouse;"

### 修改数据

update warehouse set warehouse_name='New Name' where warehouse_id=3;
 select * from warehouse;
进入mysql控制台输入 
select * from warehouse;"

### 删除数据

输入 delete from warehouse where warehouse_id=4;
输入 select * from warehouse;
进入mysql控制台输入 
 select * from warehouse;"
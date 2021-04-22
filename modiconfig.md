# UXDB实例

## 修改配置文件

UXDB数据库系统参数配置文件为uxsinodb.conf和uxsinodb.auto.conf，uxsinodb.auto.conf不建议手动进行修改，可以用alter system命令进行改写。uxsinodb.conf在单机版和MPP中均位于数据目录，(在DFS中，一般位于/home/uxdb/uxdbinstall/local_dfs 下）。

### 修改

编辑文件uxsinodb.conf，例如修改端口号、和shared_buffers

~~~shell
vi uxsinodb.conf
port = 5433
shared_buffers = 1GB 
~~~

### 生效

对系统配置文件的修改，有的参数需要重启生效。对于不需要重启生效的参数，可以用：

~~~shell
reload -D /home/uxdb/uxdbinstall/dbsql/data
~~~

重新读取配置文件或者用超级管理员在uxsql中执行：

~~~sql
select ux_reload_conf();
ux_reload_conf
\----------------
t
~~~


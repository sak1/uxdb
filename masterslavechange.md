# UXDB实例

## 判断主备角色的五种方法

进行流复制主备切换之前首先得知道当前数据库的角色，这里提供五种方法判断数据库角色，测试环境为一主一备，这些方法同样适用于一主多备环境，对于级联复制或逻辑复制的场景不完全适用 但基本思想是一致的。

#### 1、操作系统上查看WAL发送进程或WAL接收进程

之前介绍了流复制部署过程，大家知道流复制主库上有WAL发送进程，流复制备库上有WAL接收进程，根据这个思路，在数据库主机上执行以下命令

```shell
ps ef | grep "wal" | grep -v "grep"
```

如果输出wal sender .. streaming进程则说明当前数据库为主库

如果输出wal receiver  . .  streaming进程则说明当前数据库为备库

#### 2、数据库上查看WAL发送进程或WAL接收进程

在数据库层面查看WAL发送进程和WAL接收进程，例如在主库上查询问＿stat_replication视图，，如下所示：

```sql
select pid, usename, application_name, client_addr , state, sync_state FROM ux_stat_replication; 
```

如果返回记录说明是主库，备库上查询此视图无记录

同样，在备库上查看ux_stat_wal_receiver视图，如果返回记录说明是备库。

### 3、通过系统函数查看

登录数据库执行以函数，如下所示：

```sql
SELECT ux_is_in_recovery();
 ux_is_in_recovery 
 f
(1 row)
```

如果返回t说明是备库，返回f说明是主库。

#### 4、查看数据库控制信息

通过ux_controldata命令查看数据库控制信息，内容包含WAL日志信息、 checkpoint、数据块等信息，通过Database cluster state 信息可判断是主库还是备库， 如下所示：

```sql
./ux_controldata -D ../data |grep cluster
Database cluster state:               in production
```

以上查询结果返回in production表示为主库，返回in archive  recovery表示是备库

### 5、查看配置文件

```shell
vim uxsinodb.conf

synchronous_commit = on                # synchronization level;                                   
synchronous_standby_names  ='local2'   

```

如开启如上，即为主库
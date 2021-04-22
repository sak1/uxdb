# UXDB实例

## 主库上创建表空间时备库宕机

对于主备应用场景来说，由于业务或存储的变更需要，需要在主备机上做同样的操作，比如创建表空间，如果需要新部署一个项目，创建一个新数据库，需要创建一个新表空间；或者数据库新加了硬盘，需要创建新表空间指向新的硬盘。 创建表空间前需要先创建对应的表空间目录，流复制环境也是如此，只是在主库创建表空间之前需在主库、备库主机上提前创建好表空间目录，如果忘记在备库上创建表空间目录，当在主库上创建表空间时备库就会宕机。

模拟这一案例，测试环境为一主一备异步流复制环境，uxdb1为主库，uxdb2为备库：

```
select pid, usename, application_name, client_addr, state, sync_state, sync_priority from ux_stat_replication;

pid | usernamee | application_name | client_addr | state | sync_state | sync_priority
一－－－－－－－＋－－－－－－－－－＋－－－－－－－－－一－一一－－－－－＋－－－－－－－－－－－－＋－－一一－一－－－一＋－－－－－－－－－－－－＋－－－－－－－－－－－－
24924|  repuser  | node2  |192.168.138.132|  streaming |  async  |	0
(1  row ) 
```

计划在主库上新增data2表空间，在主节点uxdb1上创建表空间目录，

```
mkdir -p /home/uxdb/uxdbinstall/data2 
```

之后在主库上创建tbs_his表空间：

```sql
create tablespace data2 owner pguser location ’/home/uxdb/uxdbinstall/dbsql/data2; 
create tablespace 
```

这时，发现备库实例已经宕机

由于主库创建表空间时备库主机上没有创建相应的表空间目录，导致备库实例异常关闭，根据数据库日志提示，在备库上创建相应表空间目录，之后重启备库即可。

```
mkdir -p  /home/uxdb/uxdbinstall/data2 
```

在uxdb2上启动备库，如下所示：

```
./ux_ctl start -D ../data

waiting for server to start.................2021-04-21 10:27:13.524 CST [3865] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2021-04-21 10:27:13.524 CST [3865] LOG:  listening on IPv6 address "::", port 5432
2021-04-21 10:27:13.526 CST [3865] LOG:  listening on Unix socket "/tmp/.s.UXSQL.5432"
2021-04-21 10:27:13.532 CST [3865] LOG:  redirecting log output to logging collector process
2021-04-21 10:27:13.532 CST [3865] HINT:  Future log output will appear in directory "log".
uxmaster status ready   
 done
server started
```

这时查看备库数据库日志，无错误信息，在主库上查到WAL发送进程：

```
select pid, usename, application_name, client_addr, state, sync_state, sync_priority from ux_stat_replication;

pid | usename | application _name | client_addr | state | sync_state | sync_priority 
－－－－一－－－－＋－－－－－－ ＋－－－－－－－－－－－－－－－－＋－－－－－－－－一一一一－＋－－－－－－－－－－＋－－－－－－－－－－－＋－－－－－－－－－－－－－
24924|  repuser  | node2 |192.168.138.132|  streaming |  async |	1
(1  row ) 
```

说明异步流复制环境恢复了。
本例属于流复制生产环境案例，在主库上创建表空间时，很容易忘记事先在所有备库主机上创建表空间目录，生产系统维护操作实施前需再三核实脚本，同时做好数据库监控，如有异常及时发现并修复。
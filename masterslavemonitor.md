# UXDB实例

## 监控主备库状态与延迟

流复制部署完成之后，通常需要监控流复制主库、备库的状态，主库上主要监控WAL发送进程信息，ux_stat_replication视图显示WAL发送进程的详细信息，这个视图对于流复制的监控非常重要，如下所示：

```sql
SELECT * FROM ux_stat_replication ; 
 pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_st
art | backend_xmin | state | sent_lsn | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | re
play_lag | sync_priority | sync_state 
-----+----------+---------+------------------+-------------+-----------------+-------------+-----------
----+--------------+-------+----------+-----------+-----------+------------+-----------+-----------+---
---------+---------------+------------
(0 rows)
```

 视图中的主要字段解释如下：

- pid:  WAL发送进程的进程号。

- usename:  WAL发送进程的数据库用户名。
- application_name：连接WAL发送进程的应用别名， 此参数显示值为备库recovery.conf配置文件中primary_conninfo参数application_name选项的值。
- client_addr： 连接到WAL发送进程的客户端IP地址，也就是备库的IP。
- backend  start :  WAL发送进程的启动时间。
- state：显示WAL发送进程的状态，startup表示WAL进程在启动过程中； catchup表示备库正在追赶主库；streaming表示备库已经追赶上了主库， 并且主库向备库发送WAL日志流， 这个状态是流复制的常规状态； backup表示通过ux_basebackup正在进行备份； stopping表示WAL发送进程正在关闭。
- sent lsn:  WAL发送进程最近发送的WAL日志位置。
- write  lsn：备库最近写入的WAL日志位置，这时WAL日志流还在操作系统缓存中，还没写入备库WAL日志文件。
- flush  lsn：备库最近写入的WAL日志位置，这时WAL日志流已写入备库WAL日志文件。
- replay  lsn：备库最近应用的WAL日志位置。
- write_lag：主库上WAL日志落盘后等待备库接收WAL日志（这时WAL日志流还没写入备库WAL日志文件，还在操作系统缓存中）并返回确认信息的时间。
- flush_lag：主库上WAL日志落盘后等待备库接收WAL日志（这时WAL日志流已写入备库WAL日志文件， 但还没有应用WAL日志） 井返回确认信息的时间。
- replay_lag：主库上WAL日志落盘后等待备库接收WAL日志（这时WAL日志流已写入备库WAL日志文件，并且己应用WAL日志）并返回确认信息的时间。
-  sync _priority：基于优先级的模式中备库被选中成为同步备库的优先级， 对于基于quorum的选举模式此字段则无影响。
- sync_state ：同步状态，有以下状态值，async表示备库为异步同步模式； potential表示备库当前为异步同步模式，如果当前的同步备库岩机，异步备库可升级成为同步备库； sync表示当前备库为同步模式； quorum表示备库为quorumstandbys的候选，

### 监控主备延迟

同步流复制和异步流复制主备库之间的延迟客观存在，当流复制主库、备库机器负载较低的情况下，主备延迟通常能在毫秒级，数据库越繁忙或数据库主机负载越高主备延迟越大，有两个维度衡量主备库之间的延迟：通过WAL延迟时间衡量，通过WAL日志应用延迟量衡量，下面详细介绍。

#### 方式一：通过WAL延迟时间衡量

WAL的延迟分为write延时、 flush延时、 replay延时，分别对应ux_stat_ replication的write_lag、 flush lag、 replay_lag字段，上一节已经详细解释了这三个字段， 通过备库WAL日志接收延时和应用延时判断主备延时，在流复制主库上执行如下SQL:

```sql
select pid,usename,client_addr, state, write_lag, flush_lag, replay_lag from ux_stat_replication ; 
```

对于一个有稳定写事务的数据库，备库收到主库发送的WAL日志流后首先是写入备库主机操作系统缓存，之后写入备库WAL日志文件，最后备库根据WAL日志文件应用日志，因此这种场景下write_lag、 flush_lag和replay_lag大小关系如下所示：r ep lay _ lag  >  flush _ lag  >  write_lag 以上查询中flush_lag时间为0. 2008毫秒， replay lag时间为0. 2916毫秒，replay_l ag 
延时大于flush一lag延时很好理解，因为只有备库接收的WAL日志流写入WAL日志文件后
才能应用WAL，因此replay_lag要大于flush_lag。
write_lag、 flush_lag、 replay_lag为新版本新增字段， 2.1版本前ux_stat_ 
replication视图不提供这三个字段， 但是也有办法监控主备延时， 在流复制备库执行以下SQL，如下所示：

```
select extract (second from now() - ux_last_xact_replay_timestamp()); 
```

ux_last_ xact _replay_timestamp函数显示备库最近WAL日志应用时间， 通过与当前时间比较可粗略计算主备库延时，这种方式的优点是即使主库者掉，也可以大概判断主备延时。 缺点是如果主库上只有读操作，主库不会发送WAL日志流到备库，ux_last_xact_replay _ timestamp函数返回的结果就是一个静态的时间， 这个公式的判断结果就不严谨了。

#### 方式二：通过WAL日志应用延迟量衡量

通过流复制备库WAL的应用位置和主库本地WAL写入位置之间的WAL日志量能够准确判断主备延时，在流复制主库执行以下SQL:

```sql
select pid,usename,client_addr,state, 
ux_wal_lsn_diff(ux_current_wal_lsn() ,write_lsn) write_delay,
ux_wal_lsn_diff(ux_current_wal_lsn(),flush_lsn) flush_delay, 
ux_wal_lsn_diff(ux_current_wal_lsn(),replay_lsn) replay_dely 
from ux_stat_replication; 
```

ux_current_wal_lsn函数显示流复制主库当前WAL日志文件写入的位置，ux_wal_lsn_di ff函数计算两个WAL日志位置之间的偏移量，返回单位为字节数，以上内容显示流复制备库WAL的write延迟560字节，flush延迟896字节， replay延迟1272字节，这种方式有个缺点，当主库若掉时，此方法行不通。

#### 方式三：通过创建主备延时测算表方式

这种方法在主库上创建一张主备延时测算表，并定时往表插入数据或更新数据，之后在备库上计算这条记录的插入时间或更新时间与当前时间的差异来判断主备延时，这种方法不够严谨，但很实用， 当主库若机时，这种方式依然可以大概判断出主备延时。
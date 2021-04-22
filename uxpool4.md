# UXDB实例

==在验证中==

## 使用uxpool搭建高可用、负载均衡数据库集群架构

UXpool有很多功能，其中最重要的我觉得是如下几个：提供

+ 连接池（负载均衡模式），

+ 复制模式（能通过uxpool分发sql，因此是基于sql语句的分发复制）

+ 主备模式（依赖其他的复制，如snoly和流复制，但uxpool能把客户端的sql请求根据sql是查询还是修改发送到备库或主库），

+ 并行模式（其实就是把表水平拆分到各个数据节点，一条sql查询时需要从多个数据节点查询数据）


本文以最常用的uxdb的主备模式（主库加流复制为例来搭建，1主库+多备库，实现高可用和负载均衡）。

高可用即一个节点宕机不影响整体业务运行，负载均衡是指客户端发过来的链接请求能均匀的分布到各个数据节点，负载均衡的时候需要考虑到主库和备库是不同的，主库可读可写而备库只能读，因此select语句可以发往主库和备库，而update、insert、delete等要在主库执行，别的负载均衡软件如lvs是做不到的，但uxpool可以检测sql语句，自动发往不同的节点。

本文用uxpool-ii来实现高可用和读写分离的负载均衡。 

### 1.安装uxdb2.1，步骤略

主机名 ip 功能

uxtest5 10.1.1.14 主库

uxtest6 10.1.1.15 备库和uxpool-ii 

### 2.配置流复制：

略，流复制用户为repl用户 

### 3.下载uxpool 

uxpool_Linux7_2.1.0.2.tar.bz2

### 4.安装uxpool

上传到虚拟机里直接rpm -ivh安装即可：

~~~bash
[root@uxtest6 uxpool]# pwd
/opt/soft/uxpool
[root@uxtest6 uxpool]# ls
uxpool_Linux7_2.1.0.2.tar.bz2
[root@uxtest6 uxpool]#tar -ivh uxpool_Linux7_2.1.0.2.tar.bz2
~~~

安装完成后查看安装路径：

```bash
[root@uxtest6 uxpool]# rpm -qa|grep uxpool
uxpool-II-ux93-3.3.2-1.uxdg.x86_64
[root@uxtest6 uxpool]# rpm -ql uxpool-II-ux93-3.3.2-1.uxdg.x86_64
/etc/uxpool-II-ux93/pcp.conf
/etc/uxpool-II-ux93/uxpool.conf
/etc/uxpool-II-ux93/uxpool.conf.sample-master-slave
/etc/uxpool-II-ux93/uxpool.conf.sample-replication
/etc/uxpool-II-ux93/uxpool.conf.sample-stream
/etc/uxpool-II-ux93/pool_hba.conf 
```

默认配置文件都在/etc/uxpool-II-ux93目录下了，其中我们要配置uxpool.conf和pcp.conf

公共部分的配置：

~~~shell
listen_addresses = '*'
port = 9999
socket_dir = '/tmp'
pcp_port = 9898
pcp_socket_dir = '/tmp'

backend_hostname0 = '10.1.1.14'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/uxdb/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = '10.1.1.15'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/uxdbinstall/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

enable_pool_hba = on
pool_passwd = '123456'
authentication_timeout = 60
ssl = on
log_destination = 'stderr'               
print_timestamp = on
log_connections = on
log_hostname = on
log_statement = on
log_per_node_statement = on
log_standby_delay = 'if_over_threshold'

syslog_facility = 'LOCAL0'
syslog_ident = 'uxpool'
debug_level = 1
pid_file_name = '/var/run/uxpool/uxpool.pid'                
logdir = '/tmp'
~~~

要开启负载均衡，需要设置：

```bash
load_balance_mode = on
```

要设置主备模式，需要设置：

```shell
master_slave_mode = on
master_slave_sub_mode = 'stream'
```

要设置主库宕机好备库能自动接管主库，需要设置：

```sh
sr_check_period = 10
sr_check_user = 'repl'
sr_check_password = '123456'
delay_threshold = 10000000
health_check_period = 1
health_check_timeout = 20
health_check_user = 'uxdb'
health_check_password = '123456'
health_check_max_retries = 0
health_check_retry_delay = 1
failover_command = '/etc/uxpool-II/failover_stream.sh %d %H /uxdb/data/trigger.file'  #其中这个文件failover_stream.sh
需要定义
```

#另外并行模式需要关闭：

```bash
parallel_mode = off
```

 主库故障后，备库切换成主库的触发文件如下：

[root@uxtest6 uxpool-II-ux93]# more failover_stream.sh 

\#! /bin/sh

\# Failover command for streaming replication.

\# This script assumes that DB node 0 is primary, and 1 is standby.

\# 

\# If standby goes down, do nothing. If primary goes down, create a

\# trigger file so that standby takes over primary node.

\#

\# Arguments: $1: failed node id. $2: new master hostname. $3: path to

\# trigger file.

failed_node=$1

new_master=$2

trigger_file=$3

\# Do nothing if standby goes down.

if [ $failed_node = 1 ]; then

exit 0;

fi

\# Create the trigger file.

/usr/bin/ssh -T $new_master /bin/touch $trigger_file

exit 0;

因此，在ux的uxdb.conf中要贺uxpool参数文件的定义（ /uxdb/data/trigger.file）一致

 

配置pcp.conf,添加如下用户及密码：

uxdb:e8a48653851e28c69d0506508fb27fc5 （密码可以用ux_md5 xxx来生成）

 

### 5.配置互信

root用户，uxpool需要在主库故障后登录到备库主机上创建triger文件

过程略

 

### 6.启动uxdb及uxpool-ii

[root@uxtest6 uxpool-II-ux93]# uxpool -n -d > /tmp/uxpool.log 2>&1 &[1] 1651

 

/home/uxdb@uxtest5$psql -h uxtest6 -U uxdb -p 9999 
uxsql (9.3.1) 
Type "help" for help. 

 uxdb=# show pool_status;

....

uxdb=# show pool_nodes; 
 node_id | hostname | port | status | lb_weight | role  
 ---------+-----------+------+--------+-----------+--------- 
 0    | 10.1.1.14 | 5432 | 2   | 0.500000 | standby 
 1    | 10.1.1.15 | 5432 | 2   | 0.500000 | primary

其中status的状态意义如下：

> 0：从未使用，直接忽略
>
> 1：server已经启动，但是连接池中没有连接
>
> 2：server已经启动，并且在连接池中存在连接
>
> 3：server没有启动或者联系不上

 

我们在uxtest6上启动uxpool后，发现有30个空闲链接：

```shell
[root@uxtest6 uxpool-II-ux93]# ps -ef|grep uxpool
root    1651  1510 0 11:20 pts/0   00:00:00 uxpool -n -d
root    1652  1651 0 11:20 pts/0   00:00:00 uxpool: wait for connection request
root    1653  1651 0 11:20 pts/0   00:00:00 uxpool: wait for connection request
root    1654  1651 0 11:20 pts/0   00:00:00 uxpool: wait for connection request
...
```

 

而在uxtest5上，我们没有连接，但通过ps命令我们可以看到已经有客户端链接了（应该是uxpool连过来的）

```shell
[root@uxtest5 ~]# ps -ef|grep post]()

root    1898  1816 0 11:20 pts/0   00:00:00 su - uxdb

uxdb  1899  1898  0 11:20 pts/0   00:00:00 -bash

uxdb  1927   1  0 11:20 pts/0   00:00:00 /usr/local/uxsql/bin/uxdb

uxdb  1928  1927  0 11:20 ?     00:00:00 uxdb: startup process  recovering 00000003000000000000001C

uxdb  1929  1927  0 11:20 ?     00:00:00 uxdb: checkpointer process  

uxdb  1930  1927  0 11:20 ?     00:00:00 uxdb: writer process   

uxdb  1931  1927  0 11:20 ?     00:00:00 uxdb: stats collector process  

uxdb  1932  1927  0 11:20 ?     00:00:00 uxdb: wal receiver process  streaming 0/1C005000

uxdb  2072  1899  0 11:22 pts/0   00:00:00 psql

uxdb  2076  1927  0 11:22 ?     00:00:00 uxdb: uxdb db_test [local] idle

uxdb  2193  1927  0 11:24 ?     00:00:00 uxdb: uxdb uxdb 10.1.1.15(33372) idle

uxdb  2306  1927  0 11:26 ?     00:00:00 uxdb: uxdb uxdb 10.1.1.15(33547) idle

root    2313  2231 0 11:26 pts/1   00:00:00 grep post
```

 

测试：

先在uxtest6（主库）上插入数据，看流复制是否正常：

```bash
db_test=# select * from t1;

 id | name 

----+------

(0 rows)

db_test=# insert into t1 values (1000,'aaa');

INSERT 0 1

db_test=# select * from t1;

 id  | name 

------+------

 1000 | aaa

(1 row)

 
```

uxtest5（备库）查询：

```bash
db_test=# select * from t1;

 id  | name 

------+------

 1000 | aaa

(1 row)
```

用uxbench连接查看负载均衡是否起效：

```bash
/tmp@uxtest5$uxbench -i -F 100 -s 10 -h uxtest6 -U uxdb db_test

creating tables...

100000 of 1000000 tuples (10%) done (elapsed 0.16 s, remaining 1.44 s).

200000 of 1000000 tuples (20%) done (elapsed 0.63 s, remaining 2.52 s).

300000 of 1000000 tuples (30%) done (elapsed 1.24 s, remaining 2.90 s).

400000 of 1000000 tuples (40%) done (elapsed 2.06 s, remaining 3.09 s).

500000 of 1000000 tuples (50%) done (elapsed 2.79 s, remaining 2.79 s).

600000 of 1000000 tuples (60%) done (elapsed 4.16 s, remaining 2.77 s).

700000 of 1000000 tuples (70%) done (elapsed 7.50 s, remaining 3.21 s).

800000 of 1000000 tuples (80%) done (elapsed 8.23 s, remaining 2.06 s).

900000 of 1000000 tuples (90%) done (elapsed 10.79 s, remaining 1.20 s).

1000000 of 1000000 tuples (100%) done (elapsed 12.00 s, remaining 0.00 s).

vacuum...

set primary keys...

done.

/tmp@uxtest5$uxbench -c 25 -j 25 -M prepared -n -s 500 -T 60 -f select.sql -h uxtest6 -p 9999 -U uxdb db_test

/tmp@uxtest5$uxbench -c 25 -j 25 -M prepared -n -s 500 -T 60 -f select.sql -h uxtest6 -p 9999 -U uxdb db_test

transaction type: Custom query

scaling factor: 500

query mode: prepared

number of clients: 25

number of threads: 25

duration: 60 s

number of transactions actually processed: 70523

tps = 1174.801298 (including connections establishing)

tps = 1176.777098 (excluding connections establishing)
```

通过ps命令查看，uxtest5和uxtest6上分别由30个客户端连接

 

### 7 测试故障切换

现在我关闭主库(uxtest6)，看是否能正常切换:

目前uxtest6为主库，关闭后uxtest5会自动切换为备库。

 
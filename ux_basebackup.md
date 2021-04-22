# UXDB实例

## 异步流复制部署备库

ux_basebackup工具是对数据库==实例级物理备份==，实现对主库的在线基准备份，并自动进入备份模式进行数据库基准备份，备份完成后自动从备份模式退出，这个工具通常作为备份工具对据库进行基准备份。

ux_  base backup工具发起备份需要超级用户权限或REPLICATION权限，

注意max_wal_senders参数配置，默认10，可大一些，ux_base backup工具将消耗至少一个WAL发送进程。 

演示通过ux_base backup工具部署异步流复制。例如：

### 1 【主库】【备库】创建repuser用户

使用超级用户uxdb登录到主库uxdb1，创建流复制用户repuser，repuser需要有REPLICATION权限和LOGIN权限，如下所示：

```
CREATE  USER  repuser 
REPLICATION 
LOGIN 
CONNECTION  LIMIT  5 
ENCRYPTED  PASSWORD 'erit312'; 
```

### 2 【主库】编辑 ux_hba.conf

【主库】进入数据目录，如/home/uxdb/uxdbinstall/dbsql/data

```
#replication privilege. 
host	replication	repuser	192.168.138.131/32	md5
host	replication	repuser	192.168.138.132/32	md5
```

### 3【主库】关闭防火墙

```
systemctl stop firewalld
```

### 4【备库】上执行操作

```shell
./ux_basebackup -h 192.168.138.131 -p 5432 -U repuser -D /home/uxdb/uxdbinstall/dbsql/data2 -Fp -Xs -v -P
Password:   /*输入上面的密码erit312*/
ux_basebackup: initiating base backup, waiting for checkpoint to complete
ux_basebackup: checkpoint completed
ux_basebackup: write-ahead log start point: 4/2A000028 on timeline 1
ux_basebackup: starting background WAL receiver
483405/483405 kB (100%), 1/1 tablespace                                         
ux_basebackup: write-ahead log end point: 4/2A000130
ux_basebackup: waiting for background process to finish streaming ...
ux_basebackup: base backup completed
```

首先对数据库做一次checkpoint，基于时间点做一个全库基准备份， 全备过程中会拷贝数据文件和表空间文件到备库节点对应目录。

主要选项解析如下：

-D   备库接收主库数据的目标路径

-F  生成的备份数据格式，支持两种格式，p(plain） 格式和t(tar）格式，

- p——plain格式，备份数据和主库上的数据文件一样；
- t——tar格式，备份文件存成了tar包格式，数据文件:base.tar、表空间文件：oid.tar；

-X WAL日志备份方式，有两种可选方式， f—fetch和s—stream,

-  f——WAL日志在基准备份完成后被传送到备节点，主库上的wal_keep_ segments参数需要设置得较大，以免备份过程中产生的WAL还没发送到备节点之前被主库覆盖掉，如果出现这种情况创建基准备份会失败，这种方式下主库会启动一个基准备份WAL发送进程；
- s ——基准备份+增量日志，* 生产环境流复制部署推荐这种方式，特别是繁忙库或者是大库。

-v  详细模式，输出各阶段的日志。
-P  输出数据文件、表空间文件近似传输百分比。
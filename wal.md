# UXDB实例

## WAL日志参数、归档、清理和常用命令

待验证

### 概述

事务日志是数据库的重要组成部分，存储了数据库系统中所有更改和操作的历史，以确保数据库不会因为故障(例如掉电或其他导致服务器崩溃的故障)而丢失数据。在UXDB中，事务日志文件称为Write Ahead Log（以下简称WAL），相当于oracle中的redo日志。

### WAL日志简介

UXDB大多数数据库行为都会被记录在WAL日志中。因此，WAL日志在数据库恢复、高可用、流复制、逻辑复制等模块中扮演着极其重要的角色。当库中的数据发生变更时：

1）change发生时：先要将变更后内容计入wal buffer中，在将变更后的数据写入data buffer；

2）commit发生时：wal buffer中数据刷新到磁盘；

3）checkpoint发生时：将所有data buffer刷新到磁盘。

### WAL日志概念

WAL日志存放在 “数据目录” /ux_wal目录中

1、REDO log

Redo log通常称为重做日志，在写入数据文件前，每个变更都会先行写入到Redo log中。其用途和意义在于存储数据库的所有修改历史，用于数据库故障恢复（Recovery）、增量备份（Incremental Backup）、PITR(Point In Time Recovery)和复制（Replication）。

2、WAL segment file

为了便于管理，UXDB把事务日志文件划分为N个segment，每个segment称为WAL segment file，每个WAL segment file大小默认为16MB。

3、XLOG Record

这是一个逻辑概念，可以理解为UXDB中的每一个变更都对应一条XLOG Record，这些XLOG Record存储在WAL segment file中。UX读取这些XLOG Record进行故障恢复/PITR等操作。

4、WAL buffer

WA缓冲区，不管是WAL segment file的header还是XLOG Record都会先行写入到WAL缓冲区中，在"合适的时候"再通过WAL writer写入到WAL segment file中。

5、LSN

LSN即日志序列号Log Sequence Number。表示XLOG record记录写入到事务日志中位置。LSN的值为无符号64位整型（uint64）。在事务日志中，LSN单调递增且唯一。

6、checkpointer

checkpointer是UX中的一个后台进程，该进程周期性地执行checkpoint。当执行checkpoint时，该进程会把包含checkpoint信息的XLOG Record写入到当前的WAL segment file中，该XLOG Record记录包含了最新Redo pint的位置。

7、checkpoint

检查点checkpoint由checkpointer进程执行，主要的处理流程如下：

获取Redo point，构造包含此Redo point检查点（详细请参考Checkpoint结构体）信息的XLOG Record并写入到WAL segment file中；
刷新Dirty Page到磁盘上；
更新Redo point等信息到 ux_control 文件中。
8、REDO point

REDO point是UX启动恢复的起始点，是最后一次checkpoint启动时事务日志文件的末尾亦即写入Checkpoint XLOG Record时的位置（这里的位置可以理解为事务日志文件中偏移量）。

9、 ux_control

ux_control 是磁盘上的物理文件，保存检查点的基本信息，在数据库恢复中使用，可通过命令 ux_controldata 查看该文件中的内容。

### wal日志触发归档

#### 1、手动切换WAL日志

在日志切换这块UX的wal日志和Oracle的redo有些不一样，oracle中redo是固定几个redo日志文件，然后轮着切换去写入，因此在io高的数据库中可以看到redo切换相关的等待事件。而在UX中wal日志是动态切换，从UX9.6开始采用这种模式。和oracle不同的是，UX中这种动态wal切换步骤是这样的：单个wal日志写满(默认大小16MB，编译数据库时指定)继续写下一个wal日志，直到磁盘剩余空间不足min_wal_size时才会将旧的 WAL文件回收以便继续使用。

那么，UX怎么去手动切换WAL日志呢？

--Oracle切换redo log

```
alter system switch logfile;
```

--UXDB之后切换WAL log

```
select ux_switch_wal();
```

#### 2、wal日志写满后会自动归档

wal日志文件默认为 16MB，这个值可以在编译 UXDB 时通过参数 "--with-wal-segsize" 更改，编译则后不能修改。

#### 3、参数archive_timeout

在UXDB.conf 文件中的参数archive_timeout，

如果设置archive_timeout=60s，意思是，wal日志60s切换一次，同时会触发日志归档。

注：尽量不要把archive_timeout设置的很小，如果很小，会很消耗归档存储，因为强制归档的日志，即使没有写满，也会是默认的16M（假设wal日志写满的大小为16M）

### 清理ux_wal日志

关于UX wal日志清理，在没有开启归档的情况下：

不超过以下两个公式计算得出的个数：

(2 + checkpoint_completion_target) * checkpoint_segments + 1 或者checkpoint_segments + wal_keep_segments + 1

如果超过了max_wal_size，那么就会删除不需要的wal。

如果开启了归档，那么归档成功了，才会被清除，所以这里注意一下，如果你开启了归档，但是归档命令是失效的，那么wal目录会一直增长，不会自动删除WAL，会使得此目录被撑爆。

#### 1、什么情况下系统自动清理wal

- 做检查点的时候
- 数据库启动时，或者修改了相关参数后重启数据库。

#### 2、手动清理wal日志

可以通过缩小涉及到的函数减少wal segment的数量，也可以手动删除，如下：

```
ux_controldata
Latest checkpoint location: 16/79FF5520
Latest checkpoint’s REDO location: 16/79FF54E8
Latest checkpoint’s REDO WAL file: 00000001000000160000001E
```


这里表示16/79FF54E8检查点已经执行，已经包含在00000001000000160000001E日志文件中，那么这个日志之前的日志是可以清理的。可以使用系统命令rm清理或者ux_archivecleanup清理

--保留000000010000001600000027之后的日志

```
ux_archivecleanup /data/ux_wal/  000000010000001600000027
```

注意：ux_wal日志没有设置保留周期的参数，即没有类似mysql的参数expire_logs_days，ux_wal日志永久保留，除非shell脚步删除几天前或UX-rman备份时候设置保留策略。

### 常用命令

#### 1、查看数据库文件目录

```sql
show data_directory;
          data_directory           
-----------------------------------
 /home/uxdb/uxdbinstall/dbsql/data
(1 row)
```

#### 2、输出数据库日志目录的所有文件

ux_ls_logdir() 也是UXDB引入的函数，主要是输出数据库日志目录的所有文件

--查看日志目录所有文件

```sql
select * from ux_ls_logdir();
   name              | size  |      modification      
--------------------------------+-------+------------------------
 uxsinodb-2021-03-15_104921.log |  1621 | 2021-03-15 11:15:34+08
 uxsinodb-2021-03-15_111547.log | 10288 | 2021-03-15 18:59:28+08
 uxsinodb-2021-03-16_092233.log |  1959 | 2021-03-16 09:23:07+08
 ......
```

--查看/data目录下的文件

```
select ux_ls_dir('/data');
```

#### 3、查看WAL目录的文件情况

ux_ls_waldir()用于输出数据库WAL目录的所有文件。

- 查看文件总大小，单位是byte

```sql
select sum(size) from ux_ls_waldir();  
  sum    
-----------
 922746880
(1 row)
```

约880MB (922746880/1027/1024)了

- 查看WAL文件数量，单个wal日志文件大小默认为16MB。

```sql
select count(*) from ux_ls_waldir();
 count 
-------
    55
(1 row)
```

共有55个文件

#### 4、查看是否归档

```sql
show archive_mode;
 archive_mode 
--------------
 off
(1 row)
```

没归档呢？

#### 5、查看运行日志的配置

show logging_collector;日志收集是否启动
show log_directory;--日志输出路径
show log_filename;--日志文件名
show log_truncate_on_rotation;--当生成新的文件时如果文件名已存在，是否覆盖同名旧文件名

运行日志包括Error信息，定位慢查询SQL，数据库的启动关闭信息，checkpoint过于频繁等的告警信息。


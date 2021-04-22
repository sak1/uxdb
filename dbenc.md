# UXDB实例

## 全库加密

uxdb全库加密, 对数据库中所有的系统表、数据表、索引、视图和存储过程等进行加密处理。数据共享性高，全库加密会对系统性能会产生一定的影响。

```shell
./initdb -W -M -D ../datam
The files belonging to this database system will be owned by user "uxdb".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Enter new database encryption key: 
Data page checksums are enabled.
Using uxcrypto for full database encryption.

Enter new superuser password: 
Enter it again: 

creating directory ../datam ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

Success.
```

启动

```
./ux_ctl -D ../data stop 
/*停止正在运行的实例*/

./ux_ctl -M -D ../datam start 
/*启动加密实例*/
```

查看效果

```
vim base/12345/16384
```

压测

```
progress: 298.0 s, 43186.8 tps, lat 1.268 ms stddev 6.577
progress: 299.0 s, 45987.0 tps, lat 1.087 ms stddev 0.871
progress: 300.0 s, 37207.1 tps, lat 1.344 ms stddev 4.522
transaction type: ./test.sql
scaling factor: 1
query mode: prepared
number of clients: 50
number of threads: 50
duration: 300 s
number of transactions actually processed: 11822212
latency average = 1.265 ms
latency stddev = 9.682 ms
tps = 39404.077197 (including connections establishing)
tps = 39461.978139 (excluding connections establishing)
```

把加密和不加密场景下，通过批量插入数据，根据数据量推测性能损耗：

1-11822212/15483178=23.6%

/*15483178为同样环境下末加密的数据量，见[数据库调优简述](optsummary.md)*


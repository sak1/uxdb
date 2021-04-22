# UXDB实例

## 备库查询中止故障的处理

备库在用于分析任务、读操作时通常会将一些执行长时间的统计SQL，此时可能会出现以下错误并被中止，错误提示如：

```shell
ERROR: canceling statement due to conflict with recovery 
DETAIL: User query might have needed to see row versions that must be removed. 
```

根据报错信息，在主库上执行长时间查询过程中，由于此查询涉及的记录有可能在主库上被更新或删除，根据UXDB的MVCC机制，更新或删除的数据不是立即从物理块上删除，而是之后auto vacuum进程对老版本数据进行VACUUM，主库上对更新或删除数据的老版本进行VACUUM后，备库上也会执行这个操作，从而与备库当前查询产生冲突，导致查询被中断并抛出以上错误。

UXDB提供了配置参数来减少或避免这种情况出现

两个参数：

- max_standby_streaming_delay ：参数默认30秒，当备库执行SQL时，有可能与正在应用的WAL发生冲突，此查询如果30秒没有执行完成则被中止，注意30秒不是备库上单个查询允许的最大执行时间，是指当备库上应用WAL时允许的最大WAL延迟应用时间，因此备库上查询的执行时间有可能不到这个参数设置的值就被中止了，此参数可以设置成斗，表示当备库上的WAL应用进程与备库上执行的查询冲突时，WAL应用进程一直等待直到备库查询执行完成。

- hot_standby_ feedback ：此参数默认为off，即默认情况下备库执行查询时并不会通知主库，设置此参数为on后备库执行查询时会通知主库，当备库执行查询过程中，主库不会清理备库需要的数据行老版本，因此，备库上的查询不会被中止，然而，这种方法也会带来一定的弊端，主库上的表可能出现膨胀，主库表的膨胀程度与表上的写事务和备库上大查询的执行时间有关。

模拟下这个案例，测试环境为一主一备异步流复制环境，uxdb1为主库，uxdb1为备库，调整备库uxsinodb.conf以下参数：

为了测试，将max_standby_streaming_delay参数降低到10秒

```shell
max_standby_streaming_delay = 10s 
```

```
./ux_ctl reload -D ../data 
server signaled
```

编写update _s.sqI，主库执行uxbench压测脚本，执行时间为120秒：

```
\set v_id random(l, 1000000) 
update test_s  set flag='1' where id=:v_id; 
```

```shell
uxbench - c 8 - T 120 - d uxdb - U uxdb - n N - M prepared - f update_s.sql > update_8.out 2>&1& 
```

压力测试过程中，在备库上执行以下查询：

```
\timing 
Timing is on. 

select ux_sleep(15), count (*) from test_s; 

ERROR: canceling statement due to conflict with recovery 
DETAIL : User query might have needed to see row versions that must be removed. 
Time: 10433 . 102 ms (00 : 10. 433) 
```

以上代表统计表test_s的数据量， 同时使用了ux_sleep函数，查询执行到10秒左右时抛出错误。

### 有两种方式可以避开这一错误。

#### 方式一：调大max_standby_streaming_delay参数值

由于设置了max_standby _streaming_ delay参数为10秒， 当备库上执行查询与备库应用WAL日志产生冲突时，此SQL最多执行到10秒左右将被中止，因此可以将此参数值调大或调整成为－1 绕开这一错误。

以下将备库此参数调成60秒：同时将hot_standby_ feedback参数设置为off，调整完成后执行reload使配置生效。

```
ux_ctl reload -D ../data
server signaled 
```

之后再次开启uxbench压力测试脚本，在备库上执行以下查询：

```sql
select ux_sleep(l5) , count(*) from test_s; 
ux_sleep i count 
------------- +- ----- ---- 
i 10000000 
(1 row) 
time: 15327 . 394 ms (00:15.327) 
```

以上查询正常执行15秒未被中止。

#### 方式二：开启hot_standby_feedback参数

hot_  standby_ feedback参数设置成on后， 备库执行查询时会通知主库，备库执行大查询过程中，主库不会清理备库需要用的数据行老版本， 备库上开启此参数的代码如下：

以上设置hot_standby_feedback参数为on，同时将max_stan dby_streaming_delay参数设置为10秒，调整完成后执行reload使配置生效：

```
./ux_ctl  reload -D../data
server  signaled 
```

之后再次开启uxbench压力测试脚本， 在备库上执行以下查询，如下所示：

```
SELECT ux_sleep(l5),count(*) FROM test_s; 
ux_ sleep  |  count 
－－－－－－－－一－－－－＋－－－－－－－－－－
1  10000000 
( 1  row) 
T i me:  15349 . 958  ms  ( 00 : 15. 350) 
```

以下查询正常执行了15秒，没有被中止。

以上两种方式都可以绕开这一错误，需要注意的是

方式一——设置max_standby_ streaming_ delay参数为－1，可能造成备库上慢查询由于长时间执行而消耗大量主机资源，建议根据应用情况设置成一个较合理的值； 

方式二——开启hot_standby_feedback参数可能会使主库某些表产生膨胀

无论选择哪种方式，都应该加强对流复制主库、备库慢查询的监控，分析是否需要人工介入维护。
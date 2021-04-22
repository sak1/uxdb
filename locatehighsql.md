# UXDB实例

## 查找UXDB中CPU高负荷的SQL语句

### TOP监控

查看占用CPU高的uxdb进程，并获取该进程的ID号，如图该id号为28214

```shell
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND 
 ......
 28214 uxdb      20   0  320828  28452  25792 R  16.9  0.7   0:02.00 uxdb
 28236 uxdb      20   0  320828  11800   9140 R  15.6  0.3   0:01.88 uxdb
 28195 uxdb      20   0  320828  11840   9180 R   9.0  0.3   0:02.02 uxdb
 28154 uxdb      20   0  320828  11820   9160 S   7.6  0.3   0:01.70 uxdb
 28244 uxdb      20   0  320844  27984  25296 S   7.3  0.7   0:01.88 uxdb 
 28240 uxdb      20   0  320844  11844   9156 S   6.6  0.3   0:01.65 uxdb 
 28219 uxdb      20   0  320828  12124   9464 R   5.6  0.3   0:01.72 uxdb 
 28243 uxdb      20   0  320844  11884   9196 R   5.6  0.3   0:01.66 uxdb 
 28164 uxdb      20   0  320828  11816   9156 R   5.3  0.3   0:01.77 uxdb  
 28180 uxdb      20   0  320828  11872   9212 S   5.3  0.3   0:01.64 uxdb
 28187 uxdb      20   0  320828  11892   9232 R   5.3  0.3   0:01.62 uxdb 
```

### 查找高占用SQL

切换到uxdb用户，uxsql连接到数据库，执行如下查询语句

进程id  procpid ：28214

```
SELECT procpid, START, now() - START AS lap, current_query  FROM ( SELECT backendid, ux_stat_get_backend_pid (S.backendid) AS procpid,
ux_stat_get_backend_activity_start (S.backendid) AS START,ux_stat_get_backend_activity (S.backendid) AS current_query  FROM (SELECT
ux_stat_get_backend_idset () AS backendid) AS S) AS S WHERE current_query <> '<IDLE>' and procpid=28214 ORDER BY lap DESC;

 procpid |             start             |       lap        |                                                  current_query                                      ---------+-------------------------------+------------------+-----------------------------------------------------------------------------------------------------

   28214 | 2021-04-04 11:10:00.210575+08 | -00:00:00.000429 | insert into test values (random()*10000, random()*10000, random()*10000,md5(random()::text), CURRENT
_TIMESTAMP);
(1 row)
```

procpid：进程id 如果不确认进程ID，将上面的条件去掉，可以逐条分析
start：进程开始时间
lap：经过时间
current_query：执行中的sql

### 怎么停止正在执行的SQL？

```
SELECT ux_cancel_backend(28214);
 ux_cancel_backend 
-------------------
 t
(1 row)
```

### 查看该sql的执行计划

使用explain analyze + sql语句的格式

```sql
explain analyze insert into test values (random()*10000, random()*10000, random()*10000,md5(random()::text), CURRENT_TIMESTAMP);
          QUERY PLAN  
 Insert on test  (cost=0.00..0.05 rows=1 width=52) (actual time=0.142..0.142 rows=0 loops=1)
   ->  Result  (cost=0.00..0.05 rows=1 width=52) (actual time=0.066..0.066 rows=1 loops=1)
 Planning time: 0.058 ms
 Execution time: 0.199 ms
(4 rows)
/*lanning time: 表明了生成查询计划的时间
Execution time:表明了实际的SQL 执行时间，其中不包括查询计划的生成时间*/
```

### 分析                    

本例用于说明查高占用CPU的SQL方法，SQL合理性分析在此不做重要，本例主要是批量插入数据对CPU造成的压力。



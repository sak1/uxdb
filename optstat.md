# UXDB实例

## 统计信息和查询计划

数据库在运行期间会收集大量的数据库、表、索引的统计信息，查询优化器通过这些统计信息估计查询运行时间，然后选择最快的查询路径。 这些统计信息都保存在uxsinodb的系统表中，这些系统表都以ux_stat或ux_statio开头。 这些统计信息一类是支撑数据库系统内部方法的决策数据，例如决定何时运行autovacuum和如何解释查询计划。 这些数据保存在ux_statistics 中，这个表只有超级用户可读，普通用户没有权限，需要查看这些数据，可以从ux_stats视图中查询。 另一类统计数据用于监测数据库级、表级、语句级的信息。 

介绍几个常用的重要系统表和系统视图。

### 1、ux_stat_database 

数据库级的统计信息可以通过ux_stat _ database这个系统视图来查看，

```sql
\d ux_stat_database
```

参数说明如下：

- numbackends： 当前有多少个并发连接， 理论上控制在cpu核数的1.5倍可以获得更好的性能；
- blks_read,blks_hit： 读取磁盘块的次数与这些块的缓存命中数；
- xact_commit, xact_rollback：提交和回滚的事务数；
- deadlocks：从上次执行ux_stat_reset以来的死锁数量。

通过下面的查询可以计算缓存命中率：

```
select blks_hit::float/ (blks_read + blks_hit) as cache_hit_ratio FROM ux_stat_database WHERE datname=current_database(); 

cache_hit_ratio  
 0.978693580039249
(1 row)
```

缓存命中率是衡量1/0性能的最重要指标，它应该非常接近1， 否则应该调整shared buffers的配置，如果命中率低于99%，可以尝试调大它的值。通过下面的查询可以计算事务提交率：

```
SELECT xact_commit ::float/(xact_commit + xact_rollback) as successful_xact_ratio FROM ux_stat_database WHERE datname=current_database(); 

 successful_xact_ratio 
 0.920529801324503
(1 row)
```

事务提交率则可以知道我们应用的健康情况，它应该等于或非常接近1，否则检查是否死锁或其他超时太多。在ux_stat_database系统视图的字段中，除numbackends字段和stats_reset字段外， 其他字段的值是自从stats_reset字段记录的时间点执行ux_stat_reset()命令以来的统计信息。

建议使用者在进行优化和参数调整之后执行ux_stat_reset()命令，方便对比优化和调整前后的各项指标。有读者看到“stat” “reset” 字样的命令会心存顾虑，担心执行这条命令会影响查询计划，实际上决定查询计划的是系统表ux_statistics，它的数据是由ANALYZE命令来填充，所以不必担心执行ux_stat_reset()命令会影响查询计划。

## 2、ux_stat_user_tables 

表级的统计信息最常用的是ux_stat_user(all)_tables视图，定义如下: 

```
\d ux_stat_user_tables 
l ast_ v acuum , last_analyze： 最后一次在此表上手动执行vacuum和analyze的时间。
l a st_autovacuum , last_autoanalyze ： 最后一次在此表上被autovacuum守护程序执行
autovacuum和analyze的时间。
idx＿只an, idx _ tup _fetch：在此表上进行索引扫描的次数以及以通过索引扫描获取的行数。
seq_ scan , seq_ tup _read： 在此表上顺序扫描的次数以及通过顺序扫描读取的行数。
n_tup_in s , n tup _ upd , n_tup_del： 插入、更新和删除的行数。
n _ live _ tup, n_dead_ tup: live阳pie与deadtuple的估计数。
```

从性能角度来看，最有意义的数据是与索引VS顺序扫描有关的统计信息。 当数据库可以使用索引获取那些行时，就会发生索引扫描。 另一方面，当一个表必须被线性处理以确定哪些行属于一个集合时， 会发生顺序扫描。 因为实际的表数据存储在无序的堆中，读取行是一项耗时的操作，顺序扫描对于大表来说是成本非常高。 因此，应该调整索引定义，以便数据库尽可能少地执行顺序扫描。 索引扫描与整个数据库的所有扫描的比率可以计算如下：

```
SELECT sum(idx_scan) I ( su m(idx_scan ) + sum(seq_scan)) as idx_scan ratio FROM ux 
stat_all_tables WHERE schemaname = 'your_schema ’; 
SELECT relname,idx_scan::float/(idx_scan+seq_scan+l) a s idx_scan_ratio FROM ux 
stat_all_tables WHERE schemaname ='your_schema'ORDER BY idx_scan_ratio ASC; 
```

索引使用率应该尽可能地接近1， 如果索引使用率比较低应该调整索引。 有一些很小的表可以忽略这个比例，因为顺序扫描的成本也很低。

### 3 ux_stat_statements 

语句级的统计信息一般通过ux_stat_ statements、 uxdb日志、 auto_explain来获取。
开启ux_stat_ statements需要在uxsinodb.conf中配置

```
shared preload libraries = 'ux stat statements '
ux_stat_statements . track = all 
```

然后执行CREATEEXTENSION启用它

```
CREATE EXTENSION ux_stat_statements; 
```

ux_ stat statements视图的定义如下：

```
\d ux_stat_statements
```

ux stat_statements提供了很多维度的统计信息，最常用的是统计运行的所有查询的总的调用次数和平均的CPU时间，对于分析慢查询非常有帮助。 例如查询平均执行时间最长的3条查询，如下所示：

```
SEL ECT calls, total_time/calls AS avg _ time , left (q u ery , 80) FROM ux_ stat_ statements ORDER BY 2 DESC LIMIT3 ; 
```

通过查询ux_stat_statements视图， 可以决定先对哪些查询进行优化可获得的收益最高，对性能提升最大。 执行ux stat_ statements _reset可以重置ux_stat_ statements的统计信息。

### 4.查看SQL的执行计划

执行计划，也叫作查询计划，会显示将怎样扫描语句中用到的表， 例如使用顺序扫描
还是索引扫描等等，以及多个表连接时使用什么连接算法来把每个输入表的行连接在一起。
在uxsinodb中使用EXPLAIN命令来查看执行计划，例如：

```
EXPLAIN SELECT * FROM tbl; 
```

在EXPLAIN命令后面还可以眼上ANALYZE得到真实的查询计划，例如：

```
EXPLAIN ANALYZE SELECT * FROM tbl; 
QUERY PLAN 
```

但是需要注意的是： 使用ANALYZE选项时语句会被执行，所以在分析INSERT、UPDATE、 DELETE、 CREATETABLE AS或者EXECUTE命令的查询计划时，应该使用一个事务来执行，得到真正的查询计划后对该事务进行回滚，就会避免因为使用ANALYZE
选项而修改了数据，例如：

```
BEGIN; 
ROLLBACK 
```

在阅读查询计划时，有一个简单的原则：从下往上看，从右往左看。 例如上面的例子，从下往上看最后一行的内容是Executiontime : 0.315 ms， 是这条语句的实际执行时间是0.315ms； 往上一行的内容是Planningtime: 4.237 ms， 是这条语句的查询计划时间4.237
ms； 往上一行，一个箭头在上一行的右侧缩进处，表示先使用tbl表上的tbl_pkey进行了Index Scan ，然后到最上面一行，在tbl 表上执行了Update。 在每行计划中，都有几项值，( cost=0.00 .. xxx）预估该算子开销有多么“昂贵”。 “昂贵”按照磁盘读计算。 这里有两个数
值：第一个表示算子返回第一条结果集最快需要多少时间；第二个数值（通常更重要）表示整个算子需要多长时间。 开销预估中的第二项（rows=xxx）表示数据库预计该算子会返回多少条记录。最后一项（width=1917） 表示结果集中一条记录的平均长度（字节），由于使用了ANALYZE选项，后面还会有实际执行的时间统计。
除了ANALYZE选项，还可以使用COSTS、 BUFFERS、 TIMING、 FORMAT这些选项输出比较详细的查询计划，例如：

```
EXPLAIN (ANALYZE On，TIMING on,VERBOSE on, BUFFERS on) SELECT * FROM tbl WHERE id = 10; 
```

使用EXPLAIN的选项（ANALYZE、 COSTS、 BUFFERS、 TIMING、 VERBOSE）可以帮助开发人员获取非常详细的查询计划，但是有时候我们会有一些需要高度优化的需求，或者是一些语句的查询计划提供的信息不能完全判断语句的优劣，这时候还可以使用session级的log_xxx_stats来判断问题。 uxsinodb在initdb后， log_statement_ stats参数默认是关闭的，因为打开它会在执行每条命令的时候，执行大量的系统调用来收集资源消耗信息，所以在生产环境中也应该关闭它，一般都在session级别使用它。
在uxsinodb.conf中 有log_parser_stats、 log_planner_stats和log_statement_ stats这几个选项，默认值都是off，其中log_parser_stats和log_planner_stats这两个参数为一组，log statement_stats为一组，这两组参数不能全部同时设置为on。查看parser和planner的系统资源使用的查询计划，如下所示：

```
set client_ min_messages = log ; 
set log_parser_stats = on; 
set log_planner_stats = on; 
```

运行EXPLAIN ANALYZE查看查询计划，如下所示：

```
EXPLAIN ANALYZE select * from tbl limit 10; 
```

查看查询的系统资源使用，如下所示：

```
set client_min_messages = log ; 
set log_parser_stats = off; 
set log_planner_stats = off; 
set log_statement_stats = on; 
EXPLAIN ANALYZE select * from tbl 1imit 10; 
```

查看parser和planner的系统资源使用情况的查询计划输出内容很多，

前两行显示查询使用的用户和系统CPU时间和已经消耗的时间。

第3行显示存储设备（而不是内核缓存）中的I/O。

 第4行涵盖内存页面错误和回收的进程地址空间。 第5行显示信号和IPC消息活动。 第6行显示进程上下文切换。




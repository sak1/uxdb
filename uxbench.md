# UXDB实例

## 基准测试工具UXBENCH

### 概要

测试的目的是了解硬件的处理能力；通过调整参数优化数据库事务处理性能。

uxbench是在UXDB的基准测试的简单程序；它可能在并发的数据库会话中反复运行相同序列的 SQL 命令，并且计算平均事务率TPS（每秒的事务数）；通过uxbench会测试一种基于 TPC-B，其中在每个事务中涉及五个SELECT、UPDATE以及INSERT命令。

> 本例仅用于说明操作方法，因硬件、参数差异，测试结果不代表数据库性能

### 参数

```
uxbench -i [ other-options ] dbname
```

选项:

- -c 客户端数量，也就是并发数据库会话数量。默认为 1

- -f 指定一个自定义脚本文件，通常用不到

- -j  线程数量

- -i 初始化模式

- -r  在基准结束后，报告平均的每个命令的每语句等待时间（从客户端的角度来说是执行时间）

- -s 插入的倍数，默认是1，即插入100000条；也就是执行多少次 generate_series(1,100000)。

- -t   每个客户端运行的事务数量。默认为 10

- -T  运行测试多少多秒

  注意： -t和-T互斥，不可同时出现

### 初始化

```
./uxbench -i -F 100 -s 400 -h 192.168.138.131 -p 5432 -U uxdb uxdb
```


系统将自动创建四个表，如果同名表已经存在会被先删除

```
uxbench_accounts   	#账户表
uxbench_branches	#支行表
uxbench_history		#历史信息表
uxbench_tellers 	#出纳表	
```

在默认的情况下-s"比例因子"为 1，这些表初始包含的行数为： 

```

table                   # of rows
uxbench_branches        1
uxbench_tellers         10
uxbench_accounts        100000
uxbench_history         0
```

### 默认测试脚本（不用理这段）

uxbench执行从指定列表中随机选中的测试脚本， 它们包括带有-b的内建脚本和带有-f的用户提供的自定义脚本

默认的内建事务脚本

```
BEGIN;
UPDATE uxbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
SELECT abalance FROM uxbench_accounts WHERE aid = :aid;
UPDATE uxbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
UPDATE uxbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
INSERT INTO uxbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
END;
```

如果选择simple-update内建脚本（选项 -N），第 3 和 4 步不会被包括在事务中。
如果选择select-only内建脚本（选项 -S），只会发出SELECT。 

### 基准测试输出

```
./uxbench -c 10 -t 100 uxdb -U uxdb   # 第一个uxdb是库名，第二个是用户名
Password: 

starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1	# 有四张表，这里是1，相当于每个表的数据*1(大概10倍，并不是准确的) -s
query mode: simple	# 共三种查询方式：-M, --protocol=simple|extended|prepared
number of clients: 10	#并行客户端连接数 -c 	
number of threads: 1	# 工作线程数，多个客户端公用这些工作线程 -j
number of transactions per client: 100  # 每个客户端的事务数
number of transactions actually processed: 1000/1000  #实际处理的事务数
latency average = 3.973 ms   #平均延迟时长
tps = 2517.159806 (including connections establishing)  #包括建立联系
tps = 2529.364018 (excluding connections establishing)  #不包括建立联系
```

### 建议

多运行几次测试，看看测试结果是不是可以重现；
注意自动清理进程autovacuum对测试的影响 
-s 和 -c数量尽量一样，-c值产超过-s，否则你将主要在度量更新争夺；
准确的结论通常在另一台机器上运行uxbench，缓解自身给系统带来的资源消耗
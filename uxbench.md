# UXDB实例

## 基准测试工具UXBENCH

### TPC

TPC(Transactionprocessing Performance Council，事务处理性能委员会)；由数十家会员公司创建的非盈利组织，总部设在美国；TPC的成员主要是计算机软硬件厂家，而非计算机用户；TPC基准都是为了比较计算机系统（硬件和软件在一起）而制定的；目前有三种TPC基准规范：TPC-A，TPC-B和TPC-C；TPC-C是理事会的最新基准，TPC-A是理事会的第一个基准；其功能是制定商务应用基准程序的标准规范、性能和价格度量，并管理测试结果的发布。

### TPC-B

TPC-B根据系统每秒可执行的事务数来衡量吞吐量；1990年8月，TPC批准了第二个基准TPC-B；与TPC-A相比，TPC-B不是OLTP基准；不适用于前端OLTP测量，它用于后端数据库服务器测量；由于OLTP与数据库压力测试之间存在实质性差异，因此无法将TPC-B结果与TPC-A进行比较。TPC-B可以看作是一个数据库压力测试，其特点是：重要的磁盘输入/输出、适中的系统和应用程序执行时间、交易完整性、

### uxbench概要

uxbench是一种在UXDB上运行基准测试的简单程序；
它可能在并发的数据库会话中一遍一遍地运行相同序列的 SQL 命令，并且计算平均事务率（每秒的事务数）；
默认情况下，uxbench会测试一种基于 TPC-B 但是要更宽松的场景，其中在每个事务中涉及五个SELECT、UPDATE以及INSERT命令。
通过编写自己的事务脚本文件很容易用来测试其他情况；
测试的目的是了解硬件的处理能力；通过调整参数优化数据库事务处理性能

### 初始化

```
uxbench -i [ other-options ] dbname
```

主要选项
-i：初始化模式
-s 插入的倍数，默认是1，即插入100000条；也就是执行多少次 generate_series(1,100000)。


创建四个表，如果同名表已经存在会被先删除

```
uxbench_accounts   	#账户表
uxbench_branches		#支行表
uxbench_history		#历史信息表
uxbench_tellers 		#出纳表	
```

在默认的情况下-s"比例因子"为 1，这些表初始包含的行数为： 

```
table                   # of rows
uxbench_branches        1
uxbench_tellers          10
uxbench_accounts       100000
uxbench_history          0
```

基准测试
uxbench [ options ] dbname
重要选项:

- -c（客户端数量）
- -t（事务数量）
- -T（时间限制）
- -j（工作者线程数量）
- -f（指定一个自定义脚本文件)

### 在uxbench中执行的"事务"

uxbench执行从指定列表中随机选中的测试脚本
它们包括带有-b的内建脚本和带有-f的用户提供的自定义脚本
每一个脚本可以在其后用@指定一个相对权重，这样可以更改该脚本的抽取概率。

- -b scriptname[@weight]
- --builtin=scriptname[@weight] 
- 把指定的内建脚本加入到要执行的脚本列表中。
- @之后是一个可选的整数权重，它允许调节抽取该脚本的可能性。
- 如果没有指定，它会被设置为 1。
- 可用的内建脚本有：tpcb-like、simple-update和select-only。

### 在uxbench中的"事务"

 默认的内建事务脚本

```
BEGIN;
--1
UPDATE uxbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
--2
SELECT abalance FROM uxbench_accounts WHERE aid = :aid;
--3
UPDATE uxbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
--4
UPDATE uxbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
--5
INSERT INTO uxbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
END;
```

如果选择simple-update内建脚本（选项 -N），第 3 和 4 步不会被包括在事务中。
如果选择select-only内建脚本（选项 -S），只会发出SELECT。 



### 基准测试输出

uxbench的典型输出像这样：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10	 # 有四张表，这里是10，相当于每个表的数据*10(大概10倍，并不是准确的) -s
query mode: simple    # 共三种查询方式：-M, --protocol=simple|extended|prepared
number of clients: 10	  #并行客户端连接数 -c 	
number of threads: 1   # 工作线程数，多个客户端公用这些工作线程 -j
number of transactions per client: 1000
number of transactions actually processed: 10000/10000

tps = 85.184871 (including connections establishing)
tps = 85.296346 (excluding connections establishing)
```

前六行报告一些最重要的参数设置。
接下来的行报告完成的事务数以及预期的事务数（后者就是客户端数量与每个客户端事务数的乘积），除非运行在完成之前失败，这些值应该是相等的（在-T模式中，只有实际的事务数会被打印出来）。

最后两行报告每秒的事务数，分别代表包括和不包括开始数据库会话所花时间的情况。 
默认的类 TPC-B 事务测试要求预先设置好特定的表。
可以使用-i（初始化）选项调用uxbench来创建并且填充这些表（当你在测试一个自定义脚本时，你不需要这一步，但是需要按你自己的测试需要做一些设置工作）。

### 使用建议

多运行几次测试是一个好主意，这样可以看看你的数字是不是可以重现；
注意自动清理进程autovacuum对测试的影响 
初始化的比例因子（-s）应该至少和你想要测试的最大客户端数量一样大（-c），否则你将主要在度量更新争夺；
因为在uxbench_branches表中只有-s行，并且每个事务都想更新其中之一，因此-c值超过-s将毫无疑问地导致大量事务被阻塞来等待其他事务。 
uxbench的一个限制是在尝试测试大量客户端会话时，它自身可能成为瓶颈，可以通过在数据库服务器之外的一台机器上运行uxbench来缓解，不过必须是具有低网络延迟的机器，甚至可以在多个客户端机器上针对同一个数据库服务器并发地运行多个uxbench实例

### 示例

```sql
create database uxbenchdb
uxbench -i -s 5 uxbenchdb	--初始化，将在uxbench_accounts表中创建 500,000行。
uxbench -r -j2 -c4   -t60  uxbenchdb	--基准测试1，并行工作线程数2，客户端数量4，每客户端事务数60	
uxbench -r -j2 -c10 -T10 uxbenchdb	--基准测试2，并行工作线程数2，客户端数量10，运行时间1分钟
```

- -r   在基准结束后，报告平均的每个命令的每语句等待时间（从客户端的角度来说是执行时间）
- -j    uxbench中的工作者线程数量。在多 CPU 机器上可使用多于一个线程。客户端尽可能均匀地分布到可用的线程上。默认为 1
- -c   模拟的客户端数量，也就是并发数据库会话数量。默认为 1
- -t   每个客户端运行的事务数量。默认为 10
- -T   运行测试这么多秒，而不是为每个客户端运行固定数量的事务

注意： -t和-T互斥，不可同时出现

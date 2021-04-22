# UXDB实例

## 批量建数据的两种方法

工作中，经常需要制造一些数据，用于模拟或测试。下面我们介绍两种生成模拟数据的方法：

### 方法1、利用压测工具批量插入，适用不对时间有要求的场景，

#### 1)、UXDB下建表

```sql
create table test(  
  id int,    
  testid int,  
  personid int,  
  imgurl text, 
  rectime timestamp
);
```

##### 2）编辑写入数据语句

```sql
vim test.sql

insert into test values(random()*10000, random()*10000, random()*10000, md5(random()::text), CURRENT_TIMESTAMP); 
```

#### 3)压入数据执行

```shell
./uxbench -M prepared -n -r -P 1 -f ./test.sql -c 50 -j 50 -T 300

scaling factor: 1
query mode: prepared
number of clients: 50
number of threads: 50
duration: 300 s
number of transactions actually processed: 14731890
latency average = 1.018 ms
latency stddev = 7.049 ms
tps = 49102.447960 (including connections establishing)
tps = 49116.676602 (excluding connections establishing)
```

uxbench是基准压测工具
-M 查询模式，包括 simple: 简单协议、extended: 扩展协议、prepared: 预处理语句；
-r 基准测试运行完成后报告每个命令的平均延迟时间（执行时间）；
-n 测试运行前不执行vacuum；
-f 从指定文件中读取事务脚本；
-P 每秒显示一次进度报告。报告包括已运行时间，上次报告到当前的tps、事务延迟时间、标准误差；
-c 模拟的客户数量。具体指并发运行的数据库会话数。默认值为1；
-j 指定线程数量。在多核场景下使用多于一个线程是有意义的，由于每个线程会被分配相同数量的客户端数目，所以==客户端数量c必须设置为线程数目j 的整数倍==。该参数默认值为1。
-T 运行时长，以秒计。

自己编一下，试试吧！

#### 4）查询

```sql
select ux_size_pretty(ux_relation_size('test'));  *查询表大小
ux_size_pretty 
----------------
 1189 MB
(1 row)
```

#### 5 清理数据

```sql
drop table test;
\q
rm -rf test.sql
```

### 2、copy命令实现数据文件与数据表的导入导出，适合于对记录数量有要求的场景

copy或\copy元命令能够将一定格式的文件数据导入到数据库中，相比INSERT命令插入效率更高，大数据量的文件导入一般在数据库服务端主机由超级用户使用copy命令导入。
下面通过一个例子简单看看COPY命令的效率，测试机为一台物理机上的虚机，配置为4核CPU, 4GB内存。
首先创建一张测试表并插入一千万数据，如下所示：

```sql
create table test( 
id int4 , 
info text , 
create_time timestamp(6) with time zone default clock_timestamp()); 

insert into test(id, info) select n,n||'_c' from generate_series(1,10000000) n; 
INSERT 0 10000000
Time: 34558.955 ms (00:34.559)
/* || 是uxdb字符串连接符，与Oracle类似，与mysql的contant和+等同，此处用于组合生成info字段的模拟数据。*/
```

INSERT插入一千万数据：34秒

#### 1、数据从表导出到文件

```sql
\timing 
copy test to '/home/uxdb/test.txt'; 
COPY 10000000
Time: 12361.410 ms (00:12.361)
```

#### 2、数据从文件导入到表

一千万数据导出花了12.361秒，之后清空表test,并将文件test.txt的一千万数据导入到表中，如下所示：

```sql
truncate table test; 
copy test from '/home/uxdb/test.txt'
COPY 10000000
Time: 16018.386 ms (00:16.018)
```


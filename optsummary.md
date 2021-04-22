# UXDB实例

## 数据库调优简述

### 简述

在硬件层面，影响数据库性能的主要因素有CPU、 I/O、 内存和网络；在软件层面则要复杂得多，操作系统配置、 中间件配置、数据库参数配置、 运行在数据库之上的查询和命令等，随着业务增长、数据量的变化，应用复杂度变更等种种因素的影响，数据库系统遇到瓶颈，运行在不健康状态的情况也很多。 硬件不同，应用类型不同，优化方法也不同。

### 硬件与操作系统

#### 存储

生产环境通常使用固态磁盘，如目前使用广泛的SATASSD和PCieSSD。建议使用企业级SSD； 或使用外部存储设备，例如SAN（存储区域网络）和NAS（网络接入存储）

#### CPU

服务器在BIOS中如可设置CPU的性能模式，可能的模式有高性能模式、普通模式和节能模式，建议使用高性能模式

#### 内存

较大的内存明显降低服务器的I/O压力，缓解CPU的I/O等待时间，对数据库性能起着关键作用。 需要密切监控容量和指标趋势，在适当的时候进行扩容和升级。

####  I/O调度算法

磁盘I/O通常是数据库服务器主要瓶颈，调整调度算法提高数据库服务器性能。 对于数据库的读写操作，Linux系统在收到数据库的请求时，内核并不立即执行请求，而是通过I/O调度算法，先尝试合并请求，再发送到块设备中。

#### 预读参数

在内存中读取数据比从磁盘读取要快，增加Linux内核预读，对于大量顺序读取的操作，可有效减少I/O等待时间。

#### Swap

当内存不足时，操作系统会将虚拟内存写入磁盘进行内存交换，而数据库并不知道数据在磁盘中，这种情况下就会导致性能急剧下降，甚至造成生产故障。 有些系统管理员会彻底禁用Swap，但如果这样，一旦内存消耗完就会导致OOM（内存溢出），数据库就会随之崩溃。

#### 透明大页

透明大页（TransparentHugePages）在运行时动态分配内存，而运行时的内存分配会有延误，对于数据库来说并不友好，建议关闭透明大页。

#### NUMA架构

NUMA（Non-Uniform Memory Access）不一致内存访问。这意味着NUMA架构内存比特定CPU访问的本地内存更复杂。

### 数据库

#### 全局参数

vim uxsinodb.conf  修改相应参数，重新启动数据库服务

#### 统计信息查询计划

数据库在运行期间会收集大量的数据库、表、索引的统计信息，查询优化器通过这些统计信息估计查询运行时间，然后选择最快的查询路径。

#### 索引

索引对数据库应用性能至关重要。 针对不同的使用场景数据库有许多种索引类型来应对，例如B-tree、 Hash、 GiST、 SP-GiST、GIN、 BRIN和Bloom等等 

### 其它

 除此之外，定时进行VACUUM操作和ANALYZE操作，在业务低谷时段定时进行VACUUMFREEZE操作，及时处理慢查询，硬件的监控维护也很重要，性能调优不是一次性，也不能在性能表现已经很差的时候才去做，好的监控系统，历史数据的分析，慢查询的优化都需要不断完善，才能保障业务系统稳定高效运转。

### 实测

在本地建两个虚拟机local1,local2，分别安装了uxdb，local1做了优化，local2不动；通过300s数据压测，参考[batchinsert.md](batchinsert.md)，对比观察调参前后性能变化。

local2：未调参

```

progress: 298.0 s, 23571.1 tps, lat 2.123 ms stddev 0.731
progress: 299.0 s, 23861.0 tps, lat 2.095 ms stddev 0.879
progress: 300.0 s, 22843.2 tps, lat 2.190 ms stddev 1.679
select count(*) from test;

  count  

 6838045
(1 row)
```

local1：调参

```
progress: 298.0 s, 52168.1 tps, lat 0.958 ms stddev 0.850
progress: 299.0 s, 53477.8 tps, lat 0.935 ms stddev 0.618
progress: 300.0 s, 53453.2 tps, lat 0.936 ms stddev 0.613
uxdb=# select count(*) from test;
count 
15483178
(1 row)
```

300s数据压入测试：每秒钟处理事务量与实际数据量，提升==126%==

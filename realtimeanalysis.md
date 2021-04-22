# UXDB实例

## 大数据的实时采集与分析一例

### 应用背景

大规模数据的实时分析是信息行业常用的集事务型与分析型两种应用场景于一体的典型业务应用场景。下面我们以对GitHub日志采集与分析为例来说明UXDB在实时业务处理上的方法。

数据库：GitHub的每天访问日志作为数据源。https://www.gharchive.org，这些数据是JSON格式，包括20多种不同的数据event类型：
PushEvent ： 用户提交的Event；WatchEvent： 用户following的event；IssueCommentEvent： 用户新增加的问题单；高峰数据每天约3-4GB，约160万左右的event。

### 实时计算模型

本Demo将对数据进行实时入库，并在入库的同时进行如下计算，并且计算的结果可实时查询

- 统计每天的event总数
- 以每个GitHub项目，小时为单位，对所有event进行汇总统计
- 以每个GitHub项目，小时为单位，按event类型进行分类汇总统计
- 以每个GitHub项目，天为单位，对所有event进行汇总统计
- 以每个GitHub项目，天为单位，按event类型进行分类汇总统计
- 以每个GitHub项目，月为单位，对所有event进行汇总统计
- 以每个GitHub项目，月为单位，按event类型进行分类汇总统计

### 涉及的SQL与算法

- createegg.sql: 实时统计计算算法

- setupdb.sql： 数据模型的表和试图，以及必要的索引

- loader.py：数据加载脚本

- query.sql：用于实时查询的脚本

  这4个文件已打包到[realtimesql.rar](data/realtimesql.rar)

  

### 准备

登录CentOS

```
cd /home/uxdb/uxdbinstall/bin
mkdir demo
cd demo
mkdir data
```

把以下文件realtimesql.rar解压保存到demo目录下

### 配置和初始化数据库：

首先，我们创建一个demo数据库

```
./createdb demodb
```

初始化Demo数据库环境，添加实时统计计算

```shell
./uxsql -d demodb -f ./demo/createagg.sql  
```

```shell
./uxsql -d demodb -f ./demo/setupdb.sql    
```

### 加载数据:

####  实时获取GitHub数据:

执行命令采集保存2021年3月数据

```shell
cd data
for i in {10..31}; do wget http://data.gharchive.org/2021-03-$i-{0..23}.json.gz; done
```


#### 找开一个新的Xshell窗口，实时向数据库表插入数据

```shell
python loader.py "host='localhost' dbname=demodb user='uxdb' password='mypassword'" 2021-03-10 2021-03-10  
```

### 实时监控 运行命令进行分析统计 

```shell
export UXPASSWORD=mypassword
while [ true ]; do /home/uxdb/uxdbinstall/dbsql/bin/uxsql -d demodb -U uxdb -f query.sql ; sleep 5; done
```


#### 实时显示分析数据

```shell
Timing is on.
     created_at      |  weekday  | num_commits 
---------------------+-----------+-------------
 2021-03-10 23:00:00 | Saturday  |           1
 2021-03-10 22:00:00 | Saturday  |           7
 2021-03-10 20:00:00 | Saturday  |           3
 2021-03-10 19:00:00 | Saturday  |           3
 2021-03-10 18:00:00 | Saturday  |           7
 2021-03-10 17:00:00 | Saturday  |           3
 2021-03-10 14:00:00 | Saturday  |           4
 2021-03-10 12:00:00 | Saturday  |           2
 2021-03-10 11:00:00 | Saturday  |           7
 2021-03-10 10:00:00 | Saturday  |           5
 2021-03-10 09:00:00 | Saturday  |           1
 2021-03-10 03:00:00 | Saturday  |           3
 2021-03-10 00:00:00 | Saturday  |           5
(13 rows)

Time: 1.800 ms
```

上述内容实时刷新，代表某一时点commits的数量。可以进一步把数据代入到可视化工具中，实时呈现。

#### 修改监控项

我们根据自己的情况改写query.sql的SQL，实时监控自己感兴趣的实时计算结果

### 查看历史数据

```shell
cd /home/uxdb/uxdbinstall/dbsql/bin/demo/data
du -sh
3.3G
```

```shell
./uxsql
uxdb=#select ux_size_pretty(ux_database_size('demodb'));
ux_size_pretty 
2210 MB
```

### 结束演示，清理数据

```
uxdb@localhost# rm -rf  /home/uxdb/uxdbinstall/dbsql/bin/demo/data/*  
uxdb@localhost# cd /home/uxdb/uxdbinstall/dbsql/bin/
uxdb@localhost# ./dropdb demodb;
```

Author:Maverick

Editor:Steven 

Date:2020/03




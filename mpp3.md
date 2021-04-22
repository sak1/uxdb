# UXDB实例

## 搭建3节点UXDB MPP集群

MPP (Massively Parallel Processing)，即大规模并行处理，在数据库非共享集群中，每个节点都有独立的磁盘存储系统和内存系统，业务数据根据数据库模型和应用特点划分到各个节点上，每台数据节点通过专用网络或者商业通用网络互相连接，彼此协同计算，作为整体提供数据库服务。非共享数据库集群有完全的可伸缩性、高可用、高性能、优秀的性价比、资源共享等优势。

### 创建集群

分别在192.168.138.131、192.168.138.132、192.168.138.133创建uxdb1、uxdb2、uxdb3三个数据库

### 修改集群配置文件

```shell
shared_preload_libraries= ‘uxmpp'
```

### 启动所有节点上的集群

```sql
uxsql -p 5432 -c "CREATE EXTENSION uxmpp;"
```

### 在coordinator节点配置~/.uxpass文件

```shell
hostname:port:database:username:password

192.168.138.131:5432:uxdb:uxdb:mypassword
192.168.138.132:5432:uxdb:uxdb:mypassword
192.168.138.133:5432:uxdb:uxdb:mypassword
```

### 将work节点逐个加入到coordinnator的元数据表中

```sql
SELECT * from master_add_node('192.168.138.132', 5432);
SELECT * from master_add_node('192.168.138.133', 5432);
```

### 查看coordinator上的在线的活跃节点

```sql
SELECT * FROM master_get_active_worker_nodes();

添加worker节点：select * from master_add_node('ip', port);
删除worker节点：select master_remove_node('ip', port);
禁用worker节点：select master_disable_node('ip','port');
启用worker节点：select master_activate_node('ip', port);
查询所有节点：   select * from ux_dist_node;
```

```sql
create table t01(id int, id2 int, t text);
select create_distributed_table('t01', 'id2');
insert into t01 select id, id, lpad(id::text, 5, id::text) from generate_series(1,10000) as t(id);
```


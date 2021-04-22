# UXDB实例

## 同步流复制的实现


【备库】 修改数据目录中的uxsinodb.conf*

```
primary_conninfo= 'host=192.168.138.131 port=5432 user=repuser application_name=local2' 
```

application_name选项指定备节点的别名

【备库】 重启生效

```
cd /home/uxdb/uxdbinstall/dbsql/bin
su uxdb
./ux_ctl restart -m fast -D ../data
```

【主库 】 修改数据目录中的uxsinodb.conf  ,在192行附近

```
synchronous_commit =on 或remote apply 
synchronous_standby_names='local2'
```

【主库 】

```
./ux_ctl reload -D ../data
```

【主库 】上查看复制状态

```
SELECT usename,application_name,client_addr,sync_state FROM ux_stat_replication;

usenarne  |  application_name  |  client_addr  |  syncs_tate 
--------- - +  --------- --- ---+- -- -------+  --- ----- -
repuser  | local2	|  192.168.138.132  |  sync 
( 1  row ) 
```

视图的sync_state字段变成sync，表示主库与备库之间采用同步复制方式


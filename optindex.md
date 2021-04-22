# UXDB实例

## 索引的管理与维护

索引对数据库应用性能至关重要。 针对不同的使用场景数据库有许多种索引类型来应对，例如B-tree、 Hash、 GiST、 SP-GiST、GIN、 BRIN和Bloom等等 ，由于UXDB的MVCC内部机制，当运行大量的更新操作后，会出现“索引膨胀”的现象，这时候可以通过CREATE INDEX CONCURRENTLY在不阻塞查询和更新的情况下，在线重新创建索引，创建好新的索引之后，再删除原来有膨胀的索引，减小索引尺寸，提高查询速度。 对于主键也可以使用这种方式进行重建，重建方法如下：

```sql
CREATE UNIQUE INDEX CONCURRENTLY ON test USING btree(id); 
/*CONCURRENTLY参数并行创建索引*/
CREATE INDEX 
```

可以看到id字段上同时有两个索引test_pkey和test_id_idx，如下所示：

```sql
SELECT schemaname,relname, indexrelname, ux_r el ation_size (indexrelid) AS 
index_size, id x_scan, idx tup_read , idx_tup_fetch FROM ux_stat_user_ indexes 
WHERE indexrelname IN (SELECT indexname FROM ux indexes WHERE schemaname = 
'public 'AND tablename = 'test') ; 
```

开启事务删除主键索引，同时将第二索引更新为主键的约束，如下所示：

```sql
BEGIN; 
ALTER TABLE test DROP CONSTRAINT test_pkey; 
ALTER TABLE test ADD CONSTRAINT test_id_idx PRIMA RY KEY USING INDEX 
END;
```

检查表索引，现在只有第二索引了，如下所示：

```sql
SELECT schemaname , relname, indexrelname, ux_r el ation_ size (indexrelid ) AS 
index_size , idx scan ,idx_tup_read ,i dx_tup_fetch FROM ux_stat_user _ i ndexes WHERE 
indexrelname IN (SELECT indexname FROM ux indexes WHEREJI 
```

这样就完成了主键索引的重建，对于大规模的数据库集群，可以通过ux_repack工具进行定时的索引重建。
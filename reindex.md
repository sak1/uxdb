# UXDB实例

## 索引重建

在新版本中不必停止服务即可完成索引重建，在不影响新索引写入的前提下，让用户执行重建索引操作。

```sql
reindex index anlht_index;
```




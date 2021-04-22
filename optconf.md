# UXDB实例

## 调整全局参数

vim uxsinodb.conf  修改相应参数，重新启动数据库服务

shared_buffers：共享缓存

> 数据缓存尽量大，通常为实际内存的10％是合理的，如50000(400M)

work_mem：工作内存

> 增加work_mem提高排序操作速度。通常为实际内存的2% -4%，如81920(80M)

effective_cache_size：有效缓存

> 比如有4G内存，可以设置为3.5G(437500)

maintence_work_mem：维护工作内存

> 在CREATE INDEX, VACUUM等时用到，频率不高，消耗大，给较大内存，如512M(524288)

max_connections： 最大数据连接数

> 不能大于 实际内存 /工作内存 ，如50

random_page_cost：随机访问磁盘块的代价估计。

> 参数的默认值是4，如果使用固态磁盘，建议将它设置为比seq_page＿cost稍大即可


# UXDB实例

## 释放缓存

#### free -g

显示当前系统的内存使用情况

free命令的输出结果中，可用内存(used)如果越来越少， 通常都是因为缓存；

是否使用到Swap可以作为判断内存是否够用的一个简单标准；

只要没有使用到Swap如果想把缓存释放出来，可以使用命令：

```shell
sync
echo '1'>/proc/sys/vm/drop_caches
* 如遇Permission denied，切换到超级用户（root）下执行
```

> 生产环境释放缓存的命令要慎用


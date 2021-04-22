# UXDB实例

## 修改磁盘调度算法

磁盘I/O通常是数据库服务器主要瓶颈，调整调度算法提高数据库服务器性能。 对于数据库的读写操作，Linux系统在收到数据库的请求时，内核并不立即执行请求，而是通过I/O调度算法，先尝试合并请求，再发送到块设备中。

#### 查看当前系统支持的调度算法：

dmesg|grep -i scheduler

> 1 noop（实现简单的FIFO，基本的直接合并与排序，适合SSD）
> 2 anticipatory（延迟I/O请求，可进行临界区的优化排序）
> 3 deadline（对anticipatory缺点进行改善，降低延迟时间，适合虚拟机所在宿主机器或I/O压力较重的场景，比如数据库服务器）
> 4 cfq（均匀分配I/O带宽，公平机制，系统默认）

#### 查看磁盘sda的I/O调度算法：

```shell
cat /sys/block/sda/queue/scheduler 
noop anticipatory deadline [cfq] 
```


方括号括起来的值就是当前sda磁盘所使用的调度算法。

#### 临时修改l/O调度算法

```
echo deadline>/sys/block/sda/queue/scheduler
```

#### 长期修改l/O调度算法

shell命令修改的调度算法，在服务器重启后就会恢复到系统默认值，永久修改调度算法需要修改/etc/grub.conf文件。

如找不到，可使用下面命令修改，如：

```shell
grubby --update-kernel=ALL --args="elevator=deadline"
或
grubby --update-kernel=ALL --args="elevator=cfq"

reboot
cat /sys/block/sda/queue/scheduler 
```


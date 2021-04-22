# UXDB实例

## 修改预读磁盘扇区参数

在内存中读取数据比从磁盘读取要快，增加Linux内核预读，对于大量顺序读取的操作，可有效减少I/O等待时间。

```shell
/sbin/blockdev --getra /dev/sda
8192
```

默认为256，在较新的服务器上，可以设置到16384或更大。

### 查看

### 临时修改

```shell
/sbin/blockdev --setra 16384 /dev/sda
或
echo 16384 /sys/block/sda/queue/read_ahead_kb
```

### 长期有效

把配置追写入/etc/rc.local 文件，还可以对多块磁盘设置该值：

```shell
echo "/sbin/blockdev --setra 16384 /dev/sda /dev/sda1 /dev/sda2" >>/etc/rc.local
reboot
/sbin/blockdev --getra /dev/sda
```



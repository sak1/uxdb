# UXDB实例

## 优化内存

### 1 Swap 

内存方面，对数据库性能影响最恶劣的就是Swap了。

当内存不足时，操作系统会将虚拟内存写入磁盘进行内存交换，而数据库并不知道数据在磁盘中，这种情况下就会导致性能急剧下降，甚至造成生产故障。 有些系统管理员会彻底禁用Swap，但如果这样，一旦内存消耗完就会导致OOM（内存溢出），数据库就会随之崩溃。

因此，要么我们一方面要求有足够大的内存给数据库，另一方面，系统上线前，监控一段时间，尽可能保障不出现OOM情况，使用free -g查看内存的使用情况，Swap的used列为0，正常；

```
free -g
              total        used        free      shared  buff/cache   available
Mem:              3           0           2           0           0           3
Swap:             1           0           1
```

那么应该在有可用内存时释放己使用的Swap，释放Swap的过程实际上是先禁用Swap，再启用Swap的过程。 

使用root帐号禁用Swap

```shell
swapoff -a
```

启用Swap的命令是

```sh
swapon -a
```

重点：追求性能：关闭swap，追求安全：开启swap，两者兼顾的做法：关闭swap，提供较大的内存，监控以确保不出现内存溢出



### 2.透明大页

透明大页（TransparentHugePages）在运行时动态分配内存，而运行时的内存分配会有延误，对于数据库来说并不友好，建议关闭透明大页。

#### 查看透明大页的系统配置：

```
cat /sys/kernel/mm/transparent_hugepage/enabled 
```

==[always] madvise never==   输出中方括号包围的值就是当前值

#### 关闭透明大页

```
echo never>/sys/kernel/mm/transparent_hugepage/enabled
```

#### 永久禁用透明大页

编辑/etc/rc.local，追写入

```
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then 
echo never>/sys/kernel/mm/transparent_hugepage/enabled 
fi 
if test - f /sys/kernel/mm/transparent_hugepage/defrag; then 
echo never > /sys/kernel/mm/transparent_hugepage/defrag 
fi 
```



# UXDB实例

## 开关NUMA架构

NUMA（Non-Uniform Memory Access）不一致内存访问。这意味着NUMA架构内存比特定CPU访问的本地内存更复杂。

### 查看是否为NUMA架构

```shell
numactl --hardware
available: 2 nodes (0-1)
.....
```

如有多个nodes，即为NUMA架构

### 调优

#### 1 优化方案：uxdb在node0中执行：

> 0：内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
>
> 1：内核允许分配所有的物理内存，不管当前内存状态如何
>
> 2：内核允许分配超过所有物理内存和交换空间总和的内存

##### 查看

```
cat /proc/sys/vm/overcommit_memory
```

##### 修改

```
echo '1'>/proc/sys/vm/overcommit_memory
```

##### 永久生效

```
echo "overcommit_memory = 0" >> /etc/sysctl.conf
```

#### 3 zone_reclaim_mode

zone_reclaim_mode当区域内存不足时，允许设置强制方法回收内存。如果设置为0，则不会发生区域回收。系统中的其他区域/节点将满足分配。

> 1  = Zone reclaim on 未使用的页面缓存页，性能降低
> 2  = Zone reclaim writes dirty pages out 写入脏页，限制了进程 降低单个进程的性能，不影响其他节点上其他进程性能
> 4  = Zone reclaim swaps pages 交换页限制对本地节点的分配，除非内存策略或CPU使用配置显式重写

##### 查看

```shell
cat /proc/sys/vm/zone_reclaim_mode
```

##### 修改

uxdb在很大程度上依赖于文件系统缓存，在这种情况下，需要禁用区域回收模式。

```shell
echo '0'>/proc/sys/vm/zone_reclaim_mode
```

##### 永久生效

```shell
echo "vm.zone_reclaim_mode = 0" >> /etc/sysctl.conf
```

#### 如未关闭本架构，可以临时修改numa内存分配策略

如 interleave=all （在所有node节点进行交织分配的策略）

```shell
numactl --interleave=all /home/uxdb/uxdbinstall/dbsql/bin/ux_ctl start -D ../data
```

#### 关闭NUMA

##### 方法一：通过bios关闭

```shell
BIOS:interleave = Disable / Enable
```

##### 方法二：通过OS关闭

1、编辑 /etc/default/grub 文件，加上：numa=off

```shell
GRUB_CMDLINE_LINUX="crashkernel=auto numa=off rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet"
```

2、重新生成 /etc/grub2.cfg 配置文件：

```shell
#grub2-mkconfig -o /etc/grub2.cfg
```


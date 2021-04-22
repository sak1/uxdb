# UXDB实例

## top监控

Linux操作系统提供了非常多的性能监控工具，可以全方位监控CPU、 内存、虚拟内存、磁盘I/O、网络等各项指标，为问题排查提供了便利。常用的性能检测i具，例如top、 free、 vmstat、 iostat、 mpstat、 sar, pidstat等
top命令是最常用的性能分析工具，它可以实时监控系统状态，输出系统整体资源占用状况以及各个进程的资源占用状况。 

```
top - 08:58:35 up 4 min, 1 user, load average: 0.07, 0.26, 0.13
Tasks: 176 total, 1 running, 175 sleeping, 0 stopped, 0 zombie
%Cpu(s): 0.0 us, 0.0 sy, 0.0 ni,100.0 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st
KiB Mem : 3865308 total, 2802976 free, 307860 used, 754472 buff/cache
KiB Swap: 2097148 total, 2097148 free, 0 used. 3174236 avail Mem 
```

看晕了吧，翻译一下：

- top - 08:58:35 up 4 min ——开机时间为08:58:35， 系统已开机运行4分钟了  

- 1 user	——有1个用户登录了操作系统  

- load average: 0.07, 0.26, 0.13	——1分钟、5分钟、15分钟的系统负载分别是0.07, 0.26, 0.13  

- Tasks: 176 total	——当前一共有176 个进程，   

  1 running 	1个正在运行
  175 sleeping	 175个在睡眠，
  0 stopped	0个进程停止，
  0 zombie	0个僵尸进程

- %Cpu(s): 
  0.0 us	用户CPU占用0%
  0.0 sy	内核CPU占用	0.0%
  0.0 ni 特定优先级的进程CPU占用0.0%0.0%
  100.0 id	空闲CPU为100%
  0.0 wa	IO等待的CPU占用0.0%
  0.0 hi	硬中断CPU占用0.0%
  0.0 si	软中段CPU占用0.0%
  0.0 st  虚拟机盗取占用0.0%

- KiB Mem : 

  3865308 total	共有内存3865308k     约为3.69GB
  2802976 free	空闲内存约为2.67GB
  307860 used	 在用内存约为0.29GB
  754472 buff/cache	缓存约为0.72GB

  注：total=free+used+buff/cache

- KiB Swap: 

  2097148 total	交换区总量约为2GB
  2097148 free	空闲交换区总量约为2GB
  0 used. 在使用的交换区总量0k
  3174236 可用交换取总量约为3G

在top里我们要时刻监控第五行swap交换分区的used，如果这个数值不断变化，说明内核在不断进行内存和swap的数据交换，这是真的是内存不够用了。


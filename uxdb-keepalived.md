# UXDB实例

## UXDB+KEEPALIVED搭建数据库高可用

Keepalived 是基于VRRP协议来实现的LVS服务高可用方案，可以利用其来避免单点故障。一个LVS服务会有2台服务器运行Keepalived，一台为主服务器（MASTER），一台为备份服务器（BACKUP），对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到消息时，即主服务器已宕机，备份服务器就会接管虚拟IP，继续提供服务，从而保证了高可用性。

### Keepalived+UXDB核心配置

在keepalived的configure文件中关于UXDB部分主要有两个地方：


#### 判断UXDB是否存活脚本

vrrp_script check_alived{
	script "/home/keepalived/scripts/check_ux.sh"
	interval 5
	fall 3 # require 3 faliures for K0
}

#### failover切换脚本

{

​	smtp_alert 
​	notify_master "/home/keepalived/scripts/failover.sh"
​	notify_fault "/home/keepalived/scripts/fault.sh"
}

### 部署说明

部署包括：UXDB部署，主从流复制部署，KEEPALIVED高可用部署。

1.	实施环境
Hostname
uxdb1 192.168.138.131   
uxdb2 192.168.138.132  
Virtual ip	192.168.138.132  

keepalived版本2.0.15

操作系统配置（uxdb1、uxdb2)
两块网卡均按服务器规划设置对应静态ip，关闭NetworkManager服务并取消自启动

```
systemctl stop NetworkManager
systemctl disable NetworkManager 
```

 关闭Selinux

```
setenforce 0　　
sed -i "s/^SELINUX\=enforcing/SELINUX\=disabled/g" /etc/selinux/config
```

推荐关闭防火墙

```
systemctl stop firewalld.service
systemctl disable firewalld.service
```

操作系统参数配置（内核参数相关）

```
vim /etc/sysctl.conf

kernel.sem=50100 64128000 50100 1280
fs.file-max=7672460
fs.aio-max-nr=1048576
net.core.rmem_default=262144
net.core.rmem_max=4194304
net.core.wmem_default=262144
net.core.wmem_max=4194304
net.ipv4.ip_local_port_range=9000 65500
net.ipv4.tcp_wmem=8192 65536 16777216
net.ipv4.tcp_rmem=8192 87380 16777216
vm.min_free_kbytes=512000
vm.vfs_cache_pressure=200
vm.swappiness=20
net.ipv4.tcp_max_syn_backlog=10240
net.core.somaxconn=10240
```



```
vim /etc/security/limits.conf

soft nofile 655360
hard nofile 655360
soft nproc 655360
hard nproc 655360
soft core unlimited
hard core unlimited
soft memlock 50000000
hard memlock 50000000
```



```
vim /etc/systemd/system.conf 
DefaultLimitNOFILE=65535
DefaultLimitNPROC=65535
```


两个节点均需修改，修改完毕后重启机器  

安装集群组件
1、下载keepalived最新源码
https://www.keepalived.org/download.html
2、

(1)# tar -zxvf keepalived-2.0.15.tar.gz 

(2)# cd keepalived-2.0.15 

(3)# ./configure –-prefix=/home/keepalived

编译过程中可能提示openssl未安装
解决方法 yum install openssl-devel -y

(4)# make && make install

设置集群服务开机自启动
红字部分根据环境修改

```
vim /lib/systemd/system/keepalived.service

[Unit]
Description=LVS and VRRP High Availability Monitor
After= network-online.target syslog.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/keepalived.pid
KillMode=process
EnvironmentFile=-/home/keepalived/etc/sysconfig/keepalived
ExecStart=/home/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.targe

1;systemctl daemon-reload  重新加载
2:systemctl enable keepalived.service  设置开机自动启动
3:systemctl start keepalived.service 启动
```




数据库环境变量配置
切换到uxdb用户登录

su – uxdb

修改环境变量(默认已配置好，一般无需修改，）

--参数生效
$ source ~/.bashrc

1）	配置ux_hba.conf 
根据实际网络环境，新增加配置客户端访问权限

```
$ vim $UXDATA/ux_hba.conf 

host     replication     uxdb        192.168.75.101/0         trust
host     replication     uxdb        192.168.75.102/0         trust
host      all            all             0.0.0.0/0           md5
```

这个配置文件要根据现场的环境进行修改，如IP网段及认证方式

### 数据库配置uxsinodb.conf

$ vim uxsinodbconf

必须配置参数，可以直接追加到配置文件最后

```shell
listen_addresses = ‘*’   #根据现场配置监听网段
port=5432
max_connections = 1000
wal_level = hot_standby
max_wal_senders = 5
hot_standby = on

#可选配置参数，内存参数以服务器内存4G为例
shared_buffers = 2GB
work_mem = 8MB
maintenance_work_mem = 64MB
archive_mode = on
archive_command='test ! -f /uxbackup/archiving_active||cp %p /uxbackup/%f '
wal_keep_segments = 50
log_connections = on 
```

需要在/uxbackup下手动创建archiving_active 

```
cd /uxbackup 
touch archiving_active
```



### 从库设置

将uxdb1的数据目录复制过来

```
ux_basebackup -Fp -D /home/uxdb/uxdbinstall/dbsql/data -Xs -v -P -h node01 -p 5432 -U uxdb -R
```



### 启动备库

```
ux_ctl start -D /home/uxdb/uxdbinstall/dbsql/data -l log
```

### 集群配置（node01、node02）

1、keepalived配置文件配置
具体keepalived.conf配置详见附件
2、配置集群脚本
将三个脚本分别拷贝到两个节点的/home/keepalived/scripts目录下，（注意赋予脚本执行权限）脚本见附件script压缩包
check_ux.sh、fault.sh、failover.sh   [在此下载](data/checkuxpack.rar)

```
chmod +x  /home/keepalived/scripts/*sh
```

### 可用验证

#### 1 关闭Keepalived主节点

​    当关闭keepaalived主节点，keepalived备节点就会add出虚拟VIP，通过ip addr可以看到VIP已经漂移到备节点，并且通过系统日志可以发现触发了failover脚本，数据库进行了主备切换。

#### 2 关闭主库

​    主库关闭后，keepalived检测到主库故障，keeaplived发生故障，VIP进行切换到Keepalived备节点，并触发failover脚本，数据库进行了主备切换。

#### 3 关闭备库

​    备库关闭后，keepalived和uxdb数据库不会发生任何影响
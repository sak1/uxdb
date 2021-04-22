# UXDB实例

## 多个DIR MRC OSD的复制

### 规划

| Ip           | Dir  | Mrc  | Osd  |
| ------------ | ---- | ---- | ---- |
| 192.168.1.10 | 1    | 1    | 1    |
| 192.168.1.11 | 1    | 1    | 1    |
| 192.168.1.12 | 1    | 1    | 1    |

### 前提：

三个节点防火墙关闭，SELINUX=disabled，使用NTP将三节点时间同步。

### 正常安装UXDB，安装后，192.168.1.10上dir配置文件修改如下：

```shell
[uxdb@uxsinodb xtreemfs]$ pwd
/home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs
[uxdb@uxsinodb xtreemfs]$ vi dirconfig.properties
listen.port = 32638
http_port = 30638
listen.address = 192.168.1.10
# specify whether SSL is required
ssl.enabled = false
policy_dir = /home/uxdb/xos/xtreemfs/policies
babudb.sync = FDATASYNC
babudb.plugin.0 = /home/uxdb/xos/xtreemfs/server-repl-plugin/dir.properties
uuid = 192.168.1.10uuid
......
```

### 192.168.1.10上mrc配置文件修改如下：

```sh
[uxdb@uxsinodb xtreemfs]$ vi mrcconfig.properties
listen.port =32636
http_port = 30636
listen.address = 192.168.1.10

dir_service.host = 192.168.1.10
dir_service.port = 32638
dir_service.1.host = 192.168.1.11
dir_service.1.port = 32638
dir_service.2.host = 192.168.1.12
dir_service.2.port = 32638

policy_dir = /home/uxdb/xos/xtreemfs/policies
babudb.sync = FDATASYNC
babudb.plugin.0 = /home/uxdb/xos/xtreemfs/server-repl-plugin/mrc.properties
uuid = Default-MRC
```

###  192.168.1.10上osd修改如下： 

```sh
listen.port =32640
http_port = 30640
listen.address = 192.168.1.10

dir_service.host = 192.168.1.10
dir_service.port = 32638
dir_service.1.host = 192.168.1.11
dir_service.1.port = 32638
dir_service.2.host = 192.168.1.12
dir_service.2.port = 32638

object_dir = /home/uxdb/sf_osds/osd1/
policy_dir = /home/uxdb/xos/xtreemfs/policies
uuid = Default-OSD
```

###  192.168.1.10上dir.properties修改如下

```shell
[uxdb@uxsinodb xtreemfs]$ vi /home/uxdb/xos/xtreemfs/server-repl-plugin/dir.properties
babudb.repl.participant.0 = 192.168.1.10
babudb.repl.participant.0.port = 35676
babudb.repl.participant.1 = 192.168.1.11
babudb.repl.participant.1.port = 35676
babudb.repl.participant.2 = 192.168.1.12
babudb.repl.participant.2.port = 35676

babudb.repl.sync.n = 2
babudb.repl.backupDir =/home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs/server-repl-dir
plugin.jar = /home/uxdb/uxdbinstall/dfs/java/lib/BabuDB_replication_plugin.jar
babudb.repl.dependency.0 = /home/uxdb/uxdbinstall/dfs/java/flease/dist/Flease.jar
```

> 备注：BabuDB_replication_plugin.jar 如果没有，向开发人员索要并放入引用位置。

```shell
[uxdb@uxsinodb server-repl-plugin]$ pwd
/home/uxdb/xos/xtreemfs/server-repl-plugin
[uxdb@uxsinodb server-repl-plugin]$ vi mrc.properties

babudb.repl.participant.0 = 192.168.1.10
babudb.repl.participant.0.port = 35678
babudb.repl.participant.1 = 192.168.1.11
babudb.repl.participant.1.port = 35678
babudb.repl.participant.2 = 192.168.1.12
babudb.repl.participant.2.port = 35678

babudb.repl.sync.n = 2
babudb.repl.backupDir = /home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs/server-repl-mrc
plugin.jar = /home/uxdb/uxdbinstall/dfs/java/lib/BabuDB_replication_plugin.jar
babudb.repl.dependency.0 =/home/uxdb/uxdbinstall/dfs/java/flease/dist/Flease.jar
```

### 正常安装UXDB，安装后，192.168.1.11上dir配置文件修改如下：

```
$ vi /home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs/dirconfig.properties
listen.port = 32638
http_port = 30638
listen.address = 192.168.1.11
policy_dir = /home/uxdb/xos/xtreemfs/policies
babudb.sync = FDATASYNC
babudb.plugin.0 = /home/uxdb/xos/xtreemfs/server-repl-plugin/dir.properties
uuid = 192.168.1.11uuid
```

###  192.168.1.11上mrc配置文件修改如下

```
listen.port = 32636
http_port = 30636
listen.address = 192.168.1.11
dir_service.host = 192.168.1.10
dir_service.port = 32638
dir_service.1.host = 192.168.1.11
dir_service.1.port = 32638
dir_service.2.host = 192.168.1.12
dir_service.2.port = 32638
policy_dir = /home/uxdb/xos/xtreemfs/policies
babudb.sync = FDATASYNC 
babudb.plugin.0 = /home/uxdb/xos/xtreemfs/server-repl-plugin/mrc.properties
uuid = Default-MRC1
```

 

### 192.168.1.11上osd配置文件修改如下

```
$  vi /home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs/osdconfig.properties
listen.port = 32640
http_port = 30640
listen.address = 192.168.1.11
dir_service.host = 192.168.1.10
dir_service.port = 32638
dir_service.1.host = 192.168.1.11
dir_service.1.port = 32638
dir_service.2.host = 192.168.1.12
dir_service.2.port = 32638 
object_dir = /home/uxdb/sf_osds/osd1/
policy_dir = /home/uxdb/xos/xtreemfs/policies
uuid = Default-OSD1
```

 

### 192.168.1.11上dir.properties修改如下：

```
[uxdb@uxmydb ~]$ vi /home/uxdb/xos/xtreemfs/server-repl-plugin/dir.properties
babudb.repl.participant.0 = 192.168.1.10
babudb.repl.participant.0.port = 35676
babudb.repl.participant.1 = 192.168.1.11
babudb.repl.participant.1.port = 35676
babudb.repl.participant.2 = 192.168.1.12
babudb.repl.participant.2.port = 35676
babudb.repl.sync.n = 2
plugin.jar = /home/uxdb/uxdbinstall/dfs/java/lib/BabuDB_replication_plugin.jar
babudb.repl.dependency.0 = /home/uxdb/uxdbinstall/dfs/java/flease/dist/Flease.jar
```

 

### 192.168.1.11上mrc.properties修改如下：

```
[uxdb@uxmydb ~]$ vi /home/uxdb/xos/xtreemfs/server-repl-plugin/
babudb.repl.participant.0 = 192.168.1.10
babudb.repl.participant.0.port = 35678
babudb.repl.participant.1 = 192.168.1.11
babudb.repl.participant.1.port = 35678
babudb.repl.participant.2 = 192.168.1.12
babudb.repl.participant.2.port = 35678
babudb.repl.sync.n = 2
plugin.jar = /home/uxdb/uxdbinstall/dfs/java/lib/BabuDB_replication_plugin.jar
babudb.repl.dependency.0 =/home/uxdb/uxdbinstall/dfs/java/flease/dist/Flease.jar
```

 引用的插件如果没有，找开发人员要。

 92.168.1.12的配置可以参考192.168.1.11进行。

 

### 三个节点都配置完成之后，启动DFS

#### 在任意一个节点上查看volume，然后进行创建

```shell
[uxdb@uxsinodb ~]$ mkfs.xtreemfs 192.168.1.11/myVolume11 
```

 创建好的结果查看：

```shell
[uxdb@uxmydb ~]$ lsfs.xtreemfs 192.168.1.11
Listing all volumes of the MRC: 192.168.1.11
Volumes on 192.168.1.11:32636 (Format: volume name -> volume UUID):
​     myVolume11  ->   dd48a886-8a99-47ba-bb26-54f268197a8b
End of List.
```

 

在其他节点（192.168.1.10）上看到的效果：

```shell
[uxdb@uxsinodb ~]$ lsfs.xtreemfs 192.168.1.10
Listing all volumes of the MRC: 192.168.1.10
Volumes on 192.168.1.10:32636 (Format: volume name -> volume UUID):
​     myVolume11  ->   dd48a886-8a99-47ba-bb26-54f268197a8b
```

####  在192.168.1.11上进行挂载：

```shell
[uxdb@uxmydb ~]$ mount.xtreemfs  192.168.1.11/myVolume11  ~/xtreemfs/

[ E | 5/31 11:13:23.135 | 7fc113fff700  ] operation failed: call_id=1 errno=5 message=could not connect to 'uxdb3:32636': Host not found (non-authoritative), try again later

[ E | 5/31 11:13:23.136 | 7fc11b8ec7e0  ] Got no response from server uxdb3:32636 (Default-MRC2), retrying (infinite attempts left) (Possible reason: The server is using SSL, and the client is not.), waiting 5.0 more seconds till next attempt.

[ E | 5/31 11:13:38.133 | 7fc113fff700  ] operation failed: call_id=2 errno=5 message=could not connect to 'uxsinodb:32636': Host not found (non-authoritative), try again later

[ I | 5/31 11:13:43.172 | 7fc11b8ec7e0  ] After retrying the client succeeded to receive a response at attempt 3 from server: uxmydb:32636 (Default-MRC1)
```

####  由于网络延迟，谅解另外两个节点时报错，当重试连接到本级的时候，连接上了，也就挂载了。

####  创建成功后可以在挂载上的一个节点查看：

```shell
[uxdb@uxmydb ~]$ xtfsutil ~/xtreemfs/
Path (on volume)   /
XtreemFS file Id   dd48a886-8a99-47ba-bb26-54f268197a8b:1
XtreemFS URL     pbrpc://192.168.1.10:32638/myVolume11
Owner        uxdb
Group        uxdb
Type         volume
Available/Used Space 41 GB / 0 bytes
Num. Files/Dirs   0 / 1
Access Control p.  POSIX (permissions & ACLs)
OSD Selection p.   1000,3002
Replica Selection p. default
Default Striping p. STRIPING_POLICY_RAID0 / 1 / 128kB
Default Repl. p.   not set
Snapshots enabled  no
Selectable OSDs   Default-OSD (uxsinodb:32640)
​           Default-OSD1 (uxmydb:32640)
​           Default-OSD2 (uxdb3:32640)
```

 

#### 照着上面的配置，在三个配置好一点的节点上重新做实验如下：

挂载并查看：

```shell
[uxdb@localhost ~]$ mount.xtreemfs 10.1.101.160/volume160 xtreemfs/
[uxdb@localhost ~]$ xtfsutil ~/xtreemfs/
Path (on volume)   /
XtreemFS file Id   a3732d63-c7d6-4e44-b541-8faf1d0ef988:1
XtreemFS URL     pbrpc://10.1.101.161:32638/volume160
Owner        uxdb
Group        uxdb
Type         volume
Available/Used Space 20 GB / 0 bytes
Num. Files/Dirs   0 / 1
Access Control p.  POSIX (permissions & ACLs)
OSD Selection p.   1000,3002
Replica Selection p. default
Default Striping p. STRIPING_POLICY_RAID0 / 1 / 128kB
Default Repl. p.   not set
Snapshots enabled  no
Selectable OSDs   Default-OSD160 (10.1.101.160:32640)
​           Default-OSD161 (10.1.101.161:32640)
​           Default-OSD162 (10.1.101.162:32640)
[uxdb@localhost ~]$ ls
```

 

#### 新建了一个16k大小的文件并拷贝到挂载目录中，花费时间超过半分钟：

```shell
[uxdb@localhost ~]$ cp xx.txt ./xtreemfs/ 
[uxdb@localhost ~]$ xtfsutil ~/xtreemfs/xx.txt
Path (on volume)   /xx.txt
XtreemFS file Id   a3732d63-c7d6-4e44-b541-8faf1d0ef988:2
XtreemFS URL     pbrpc://10.1.101.161:32638/volume160/xx.txt
Owner        uxdb
Group        uxdb
Type         file
Replication policy  none (not replicated)
XLoc version     0
Replicas:
 Replica 1
   Striping policy   STRIPING_POLICY_RAID0 / 1 / 128kB
   OSD 1        Default-OSD162 (10.1.101.162:32640)
```

 经查看发现，我操作机子是10.1.101.160，发现时间拷贝到了 162机子上。也证明了，拷贝文件会分布到其他节点的osd上。

####  修改复制模式：

```shell
[uxdb@localhost ~]$ xtfsutil -r WqRq ~/xtreemfs/xx.txt 
Changed replication policy to: WqRq

[uxdb@localhost ~]$ xtfsutil ~/xtreemfs/xx.txt 
Path (on volume)   /xx.txt
XtreemFS file Id   a3732d63-c7d6-4e44-b541-8faf1d0ef988:2
XtreemFS URL     pbrpc://10.1.101.161:32638/volume160/xx.txt
Owner        uxdb
Group        uxdb
Type         file
Replication policy  WqRq
XLoc version     1
Replicas:
 Replica 1
   Striping policy   STRIPING_POLICY_RAID0 / 1 / 128kB
   OSD 1        Default-OSD162 (10.1.101.162:32640)
```

 可以看到复制模式已经变为 WqRq

####  检查可以用于创建一个新文件的副本OSD列表

```
[uxdb@localhost ~]$ xtfsutil -l ~/xtreemfs/xx.txt 
OSDs suitable for new replicas: 
 Default-OSD160 (10.1.101.160:32640)
 Default-OSD161 (10.1.101.161:32640)
```

####  添加可用于创建文件副本的OSD

```
[uxdb@localhost ~]$ xtfsutil  --add-replica=auto  ~/xtreemfs/xx.txt 
Added new replica on OSD: Default-OSD161
```

####  针对文件夹设置备份属性

```
[uxdb@localhost ~]$ xtfsutil --set-drp --replication-policy WqRq --replication-factor 3 ~/xtreemfs
Updated default replication policy to: WQRQ with 3 replicas
```

 再次查看：

```shell
[uxdb@localhost ~]$ xtfsutil ~/xtreemfs/
Path (on volume)   /
XtreemFS file Id   a3732d63-c7d6-4e44-b541-8faf1d0ef988:1
XtreemFS URL     pbrpc://10.1.101.161:32638/volume160
Owner        uxdb
Group        uxdb
Type         volume
Available/Used Space 20 GB / 36 bytes
Num. Files/Dirs   1 / 1
Access Control p.  POSIX (permissions & ACLs)
OSD Selection p.   1000,3002
Replica Selection p. default
Default Striping p. STRIPING_POLICY_RAID0 / 1 / 128kB
Default Repl. p.   WqRq with 3 replicas
Snapshots enabled  no
Selectable OSDs   Default-OSD160 (10.1.101.160:32640)
​           Default-OSD161 (10.1.101.161:32640)
​           Default-OSD162 (10.1.101.162:32640)
```

####  再次将一个文件复制进目录：

```
[uxdb@localhost ~]$ cp yy.txt ~/xtreemfs/
[uxdb@localhost ~]$ xtfsutil ~/xtreemfs/yy.txt 

Path (on volume)   /yy.txt
XtreemFS file Id   a3732d63-c7d6-4e44-b541-8faf1d0ef988:3
XtreemFS URL     pbrpc://10.1.101.160:32638/volume160/yy.txt
Owner        uxdb
Group        uxdb
Type         file
Replication policy  WqRq
XLoc version     0
Replicas:
 Replica 1
   Striping policy   STRIPING_POLICY_RAID0 / 1 / 128kB
   OSD 1        Default-OSD160 (10.1.101.160:32640)
 Replica 2
   Striping policy   STRIPING_POLICY_RAID0 / 1 / 128kB
   OSD 1        Default-OSD162 (10.1.101.162:32640)
 Replica 3
   Striping policy   STRIPING_POLICY_RAID0 / 1 / 128kB
   OSD 1        Default-OSD161 (10.1.101.161:32640)
```

 鸣谢：Steven Wu
# UXDB实例

## 一个DIR MSC多个OSD的复制

### 配置前准备

#### 规划

目标：一个dir,mrc，6个osd ,配置replication
虚拟机： 192.168.1.11
加挂了一个磁盘，分为两个区 /uxdisk/disk1   和 /uxdisk/disk2
其中  disk1上为osd1  osd2  osd3
disk2上为osd4  osd5  osd6

#### 安装uxdb

（略）

#### 环境变量.bash_profile设置

```
export JAVA_HOME=/home/uxdb/jdk1.8.0_20
export PATH=$JAVA_HOME/bin:$PATH
export DFS_DIR=$HOME/uxdbinstall/dfs
export DFSURL="192.168.1.11:32638"
export DFSDB="myVolume"

PATH=$PATH:$HOME/bin
PATH=$PATH:$HOME/uxdbinstall/dbsql/bin
PATH=$PATH:$DFS_DIR/bin

export PATH
```

设置好环境变量是为后续使用方便，不用找到目录下再执行相关命令。

### 同一台机器下多个OSD配置

#### OSD配置文件修改

安装好uxdb后复制了5个osdconfig.properties配置文件，重命名后依次为：

- osdconfig1.properties
- osdconfig2.properties
- osdconfig3.properties
- osdconfig4.properties
- osdconfig5.properties
- osdconfig6.properties

修改osd配置文件参数

```
[uxdb@uxmydb xtreemfs]$ pwd
/home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs
[uxdb@uxmydb xtreemfs]$ vi osdconfig1.properties
listen.port = 32641
http_port = 30641
dir_service.host = 192.168.1.11
dir_service.port = 32638
object_dir = /uxdisk/disk1/osd1/obj
policy_dir = /uxdisk/disk1/osd1/policies
uid = Default-OSD1

[uxdb@uxmydb xtreemfs]$ vi osdconfig2.properties
listen.port = 32642
http_port = 30642
dir_service.host = 192.168.1.11
dir_service.port = 32638
object_dir = /uxdisk/disk1/osd2/obj
policy_dir = /uxdisk/disk1/osd2/policies
uid = Default-OSD2

……
```

其他4个OSD的配置以此类推

其他4个OSD的配置以此类推

#### 2.2 启动文件修改

修改启动文件添加了对6个osd配置文件的启动：

```
java -Xmx1024m -XX:MaxNewSize=128m -XX:MaxPermSize=256m  -ea -cp ./java/servers/dist/XtreemFS.jar:./java/lib/BabuDB.jar:./java/flease/dist/Flease.jar:./java/lib/protobuf-java-2.5.0.jar:./java/foundation/dist/Foundation.jar:./java/lib/jdmkrt.jar:./java/lib/jdmktk.jar:./java/lib/commons-codec-1.3.jar org.xtreemfs.osd.OSD etc/xos/xtreemfs/osdconfig1.properties 2>null &

java -Xmx1024m -XX:MaxNewSize=128m -XX:MaxPermSize=256m  -ea -cp ./java/servers/dist/XtreemFS.jar:./java/lib/BabuDB.jar:./java/flease/dist/Flease.jar:./java/lib/protobuf-java-2.5.0.jar:./java/foundation/dist/Foundation.jar:./java/lib/jdmkrt.jar:./java/lib/jdmktk.jar:./java/lib/commons-codec-1.3.jar org.xtreemfs.osd.OSD etc/xos/xtreemfs/osdconfig2.properties 2>null &
```

……

#### 2.3 Volume创建

启动DFS:

```
[uxdb@uxmydb dfs]$ pwd
/home/uxdb/uxdbinstall/dfs
[uxdb@uxmydb dfs]$ ll
total 24
drwxrwx---. 2 uxdb uxdb 4096 May 17 15:00 bin
drwxrwx---. 3 uxdb uxdb 4096 May 17 15:00 etc
drwxrwx---. 6 uxdb uxdb 4096 May 17 15:00 java
-rw-rw-r--. 1 uxdb uxdb  104 May 18 10:29 null
-rwxrwxr-x. 1 uxdb uxdb 3081 May 17 16:37 start-all.sh
-rwxrwx---. 1 uxdb uxdb 1092 May 17 15:00 stop-all.sh

[uxdb@uxmydb dfs]$ ./start-all.sh 
Starting DFS engine...
Done!
```

查看volume

```
[uxdb@uxmydb xtreemfs]$ lsfs.xtreemfs 192.168.1.11
```

新建：

```
[uxdb@uxmydb xtreemfs]$ mkfs.xtreemfs 192.168.1.11/myVolume
```

完成之后查看：

```
[uxdb@uxmydb xtreemfs]$ lsfs.xtreemfs 192.168.1.11
Listing all volumes of the MRC: 192.168.1.11
Volumes on 192.168.1.11:32636 (Format: volume name -> volume UUID):
	myVolume	->	f161b6fc-ff23-436d-bd61-b5cfbe4d88b3
End of List.
```

在家目录下新建一个目录，进行挂载

```
[uxdb@uxmydb ~]$ mkdir  ~/xtreemfs

[uxdb@uxmydb ~]$ mount.xtreemfs 192.168.1.11/myVolume  ~/xtreemfs
```

对挂载好的目录进行查看：

```
[uxdb@uxmydb ~]$ xtfsutil ~/xtreemfs
Path (on volume)     /
XtreemFS file Id     f161b6fc-ff23-436d-bd61-b5cfbe4d88b3:1
XtreemFS URL         pbrpc://uxmydb:32638/myVolume
Owner                uxdb
Group                uxdb
Type                 volume
Available/Used Space 4 GB / 135 MB
Num. Files/Dirs      1736 / 29
Access Control p.    POSIX (permissions & ACLs)
OSD Selection p.     1000,3002
Replica Selection p. default
Default Striping p.  STRIPING_POLICY_RAID0 / 1 / 128kB
Default Repl. p.     default
Snapshots enabled    no
Selectable OSDs      Default-OSD1 (192.168.1.11:32641)
                     Default-OSD2 (192.168.1.11:32642)
                     Default-OSD3 (192.168.1.11:32643)
                     Default-OSD4 (192.168.1.11:32644)
                     Default-OSD5 (192.168.1.11:32645)
                     Default-OSD6 (192.168.1.11:32646)
```



### Replication配置

#### 3.1 修改复制模式

在~/ xtreemfs 中新建一个文件 movie.avi
修改movie.avi的复制模式

```
xtfsutil -r WqRq ~/xtreemfs/movie.avi
```

说明：WqRq是一种投票表决的选择策略，正常情况下在WqRq时，至少factor为3，这样投票的时候才会出现少数服从多数的情况。设置为3，是自己本身一份数据+2个backup

#### 3.2 检查副本OSD列表

检查可以用于创建一个新文件的副本OSD列表

```
 [uxdb@uxmydb ~]$ xtfsutil -l ~/xtreemfs/movie.avi
OSDs suitable for new replicas: 
  Default-OSD3 (192.168.1.11:32643)
  Default-OSD6 (192.168.1.11:32646)
  Default-OSD5 (192.168.1.11:32645)
  Default-OSD1 (192.168.1.11:32641)
  Default-OSD2 (192.168.1.11:32642)
  Default-OSD4 (192.168.1.11:32644)
```



#### 3.3 添加可用于创建文件副本的OSD

```
xtfsutil --add-replica=auto  ~/xtreemfs/movie.avi
```

自动添加一个可用于创建文件副本的OSD，也可添加制定UUID的OSD

#### 设置文件夹备份属性

```
xtfsutil --set-drp --replication-policy WqRq --replication-factor 3  ~/xtreemfs
```

设置完成后进行查看：

```
[uxdb@uxmydb ~]$ xtfsutil ~/xtreemfs
Path (on volume)     /
XtreemFS file Id     f161b6fc-ff23-436d-bd61-b5cfbe4d88b3:1
XtreemFS URL         pbrpc://uxmydb:32638/myVolume
Owner                uxdb
Group                uxdb
Type                 volume
Available/Used Space 4 GB / 135 MB
Num. Files/Dirs      1736 / 29
Access Control p.    POSIX (permissions & ACLs)
OSD Selection p.     1000,3002
Replica Selection p. default
Default Striping p.  STRIPING_POLICY_RAID0 / 1 / 128kB
Default Repl. p.     WqRq with 3 replicas
Snapshots enabled    no
Selectable OSDs      Default-OSD1 (192.168.1.11:32641)
                     Default-OSD2 (192.168.1.11:32642)
                     Default-OSD3 (192.168.1.11:32643)
                     Default-OSD4 (192.168.1.11:32644)
                     Default-OSD5 (192.168.1.11:32645)
                     Default-OSD6 (192.168.1.11:32646)
```


标注部分表示3份复制备份，在没有配置之前是default

### 数据库相关

#### 4.1 初始化数据库

```
[uxdb@uxmydb ~]$ initdb -Z -W -D uxdb01
init local dfs root:/home/uxdb/uxdbinstall/local_dfs
The files belonging to this database system will be owned by user "uxdb".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory uxdb01 ... ok
creating subdirectories ... ok
selecting default max_connections ... 10
selecting default shared_buffers ... 400kB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
creating template1 database in uxdb01/base/1 ... ok
initializing default authentication ... ok
Enter new superuser password: 
Enter it again: 
setting password ... ok
initializing dependencies ... ok
creating system views ... ok
loading system objects' descriptions ... ok
creating collations ... ok
creating conversions ... ok
creating dictionaries ... ok
setting privileges on built-in objects ... ok
creating information schema ... ok
loading PL/uxSQL server-side language ... ok
vacuuming database template1 ... ok
copying template1 to template0 ... ok
creating UXDB as default administrative connection database ... ok

Success. You can now start the database server using:
```



    uxdb -Z -B 1024 -N 100  -D uxdb01
or

```
   ux_ctl -Z -o "-Z -B 1024 -N 100" -D uxdb01 -l logfile start
```

and
start client using:

```
uxsql -d uxdb
```



#### 4.2 启动DBserver

```
[uxdb@uxmydb ~]$ ux_ctl -Z -o "-Z -B 1024 -N 100" -D uxdb01 -l logfile start
server starting
```

查看db的运行状态

```
[uxdb@uxmydb ~]$ ux_ctl status -Z -D uxdb01
ux_ctl: server is running (PID: 4380)
/home/uxdb/uxdbinstall/dbsql/bin/uxdb "-D" "uxdb01" "-Z" "-B" "1024" "-N" "100"
```



#### 4.3 创建一个数据库

```
[uxdb@uxmydb ~]$ createdb uxdbreplica
```



#### 4.4 初始化数据

```
[uxdb@uxmydb ~]$ uxbench -i -s 6 uxdbreplica
```

#### 登录控制台

……

### 附录

新增数据./uxsql -d uxdbreplica登录控制台执行 createdata.sql；

```
uxbench -c 16 -j 2 -T 1800 -P 20 -S uxdbreplica;
Password: 
starting vacuum...end.
progress: 144.0 s, 0.0 tps, lat -nan ms stddev -nan
progress: 160.0 s, 58.7 tps, lat 265.394 ms stddev 127.350
progress: 180.0 s, 84.3 tps, lat 189.910 ms stddev 119.605
progress: 200.0 s, 106.3 tps, lat 150.641 ms stddev 110.219
progress: 220.0 s, 154.4 tps, lat 103.562 ms stddev 67.405
progress: 240.0 s, 171.1 tps, lat 93.598 ms stddev 57.201
progress: 260.0 s, 214.7 tps, lat 74.497 ms stddev 53.610
progress: 280.0 s, 272.5 tps, lat 58.729 ms stddev 38.601
progress: 300.0 s, 488.9 tps, lat 32.763 ms stddev 29.256
progress: 320.0 s, 885.3 tps, lat 18.076 ms stddev 13.898
progress: 340.0 s, 1200.4 tps, lat 13.326 ms stddev 4.749
progress: 360.0 s, 1177.8 tps, lat 13.583 ms stddev 4.126
progress: 380.0 s, 1197.6 tps, lat 13.357 ms stddev 5.326
progress: 400.0 s, 1251.9 tps, lat 12.780 ms stddev 3.533
progress: 420.0 s, 1250.3 tps, lat 12.795 ms stddev 3.336
progress: 440.0 s, 1133.7 tps, lat 14.110 ms stddev 4.360
progress: 460.0 s, 1230.9 tps, lat 12.996 ms stddev 3.559
progress: 480.0 s, 1219.9 tps, lat 13.113 ms stddev 3.532
progress: 500.0 s, 1187.4 tps, lat 13.473 ms stddev 3.968
progress: 520.0 s, 1235.0 tps, lat 12.955 ms stddev 3.635
progress: 540.0 s, 1146.6 tps, lat 13.951 ms stddev 4.180
progress: 560.0 s, 1146.7 tps, lat 13.950 ms stddev 4.729
progress: 580.0 s, 1135.2 tps, lat 14.091 ms stddev 4.219
progress: 600.0 s, 1122.8 tps, lat 14.249 ms stddev 4.859
progress: 620.0 s, 1241.1 tps, lat 12.890 ms stddev 3.381
progress: 640.0 s, 1184.2 tps, lat 13.509 ms stddev 4.885
progress: 660.0 s, 1257.4 tps, lat 12.723 ms stddev 3.249
progress: 680.0 s, 1147.4 tps, lat 13.943 ms stddev 4.140
progress: 700.0 s, 1126.6 tps, lat 14.199 ms stddev 4.672
progress: 720.0 s, 1164.6 tps, lat 13.737 ms stddev 3.846
progress: 740.0 s, 1035.7 tps, lat 15.445 ms stddev 5.197
progress: 760.0 s, 1125.0 tps, lat 14.221 ms stddev 4.828
……
```

中间kill掉了一个osd



鸣谢：Steven Wu
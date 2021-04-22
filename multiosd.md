# UXDB实例

## 不同机器上多个OSD的配置

### 配置计划

192.168.1.10   配置DIR  MRC  OSD  和OSD1  DB位于这个节点上。
192.168.1.11  配置OSD2

前提： 安装好jkd，确保两个节点防火墙关闭、两个节点时间同步。

192.168.1.10上的操作：
安装UXDB，安装过程中先不启动DFS,安装完成后到如下目录修改配置文件：

```
[uxdb@uxsinodb xtreemfs]$ pwd
/home/uxdb/uxdbinstall/dfs/etc/xos/xtreemfs
[uxdb@uxsinodb xtreemfs]$ ll
total 32
-rwxrwx---. 1 uxdb uxdb 5433 May 15 03:53 dirconfig.properties
-rwxrwx---. 1 uxdb uxdb 7198 May 15 23:24 mrcconfig.properties
-rwxrwx---. 1 uxdb uxdb 4611 May 16 16:20 osdconfig1.properties
-rwxrwx---. 1 uxdb uxdb 4616 May 15 23:25 osdconfig.properties
----------. 1 uxdb uxdb    0 May 15 03:49 sed6oD3m2
----------. 1 uxdb uxdb    0 May 15 03:49 sedAIfnwn
----------. 1 uxdb uxdb    0 May 15 03:49 sedVe63ly
```

将当前目录下的所有文件中的  etc替换为 home/uxdb

```
sed -i "s/etc/home\/uxdb/g" `grep etc -rl .`
```

将tmp也替换为home/uxdb

```
sed -i "s/tmp/home\/uxdb/g" `grep tmp -rl .`
```

（因为对uxdb来说，无tmp目录访问权限）

将osdconfig.properties复制出一份为 osdconfig1.properties
修改osdconfig1.properties内容：

```
listen.port = 32641 （区别于osdconfig.properties配置文件中的端口号）
http_port = 30641
dir_service.host = 192.168.1.10
dir_service.port = 32638
object_dir = /uxdbdisk/uxdbdata/objs
policy_dir = /uxdbdisk/uxdbdata/policies
uuid = Default-OSD1
```

将osdconfig.properties中有localhost的也换为当前节点ip

```
dirconfig.properties   mrcconfig.properties不是ip的也做同样替换。
```

修改DFS启动文件，将新加入的OSD1加入到启动文件中：

```
[uxdb@uxsinodb dfs]$ pwd
/home/uxdb/uxdbinstall/dfs
[uxdb@uxsinodb dfs]$ ll
total 24
drwxrwx---. 2 uxdb uxdb 4096 May 15 03:49 bin
drwxrwx---. 3 uxdb uxdb 4096 May 15 03:49 etc
drwxrwx---. 6 uxdb uxdb 4096 May 15 03:49 java
-rw-rw-r--. 1 uxdb uxdb  104 May 16 16:24 null
-rwxrwxr-x. 1 uxdb uxdb 1601 May 16 16:22 start-all.sh
-rwxrwx---. 1 uxdb uxdb 1092 May 15 03:49 stop-all.sh
```

```
[uxdb@uxsinodb dfs]$ vi start-all.sh 
java -Xmx1024m -XX:MaxNewSize=128m -XX:MaxPermSize=256m  -ea -cp ./java/servers/dist/XtreemFS.jar:./java/lib/BabuDB.jar:./java/flease/dist/Flease.jar:./java/lib/protobuf-java-2.5.0.jar:./java/foundation/dist/Foundation.jar:./java/lib/jdmkrt.jar:./java/lib/jdmktk.jar:./java/lib/commons-codec-1.3.jar org.xtreemfs.osd.OSD etc/xos/xtreemfs/osdconfig1.properties 2>null &
```

关闭文件不用管，因为关闭是一个循环，可以关闭多个osd

修改配置文件：

```
[uxdb@uxsinodb ~]$ pwd
/home/uxdb
[uxdb@uxsinodb ~]$ vi .bash_profile
export JAVA_HOME=/home/uxdb/jdk1.8.0_20
export PATH=$JAVA_HOME/bin:$PATH
export DFS_DIR=$HOME/uxdbinstall/dfs
export DFSURL="192.168.1.10:32638"
export DFSDB="demo"

PATH=$PATH:$HOME/bin
PATH=$PATH:$HOME/uxdbinstall/dbsql/bin
PATH=$PATH:$DFS_DIR/bin
export PATH
```

192.168.1.11上的操作：
安装UXDB，安装过程中先不启动DFS,安装完成后到如下目录修改配置文件：

将dirconfig.properties        mrcconfig.properties         osdconfig.properties  配置文件全部删除。
同时从192.168.1.10 上拷贝一份osdconfig.properties 到该目录并重命名为osdconfig2.properties ，查看如下：

```
[uxdb@uxmydb xtreemfs]$ ll
total 8
-rwxrwx---. 1 uxdb uxdb 4617 May 15 11:56 osdconfig2.properties
----------. 1 uxdb uxdb    0 May 15 11:44 sedEI31nr
----------. 1 uxdb uxdb    0 May 15 11:44 sedjoTZ3s
----------. 1 uxdb uxdb    0 May 15 11:43 sedMUXBGJ
```

将dirconfig.properties        mrcconfig.properties         osdconfig.properties  配置文件全部删除。
同时从192.168.1.10 上拷贝一份osdconfig.properties 到该目录并重命名为osdconfig2.properties ，查看如下：

修改启动文件：

```
[uxdb@uxmydb dfs]$ pwd
/home/uxdb/uxdbinstall/dfs
[uxdb@uxmydb dfs]$ ll
total 24
drwxrwx---. 2 uxdb uxdb 4096 May 15 11:43 bin
drwxrwx---. 3 uxdb uxdb 4096 May 15 11:43 etc
drwxrwx---. 6 uxdb uxdb 4096 May 15 11:43 java
-rw-rw-r--. 1 uxdb uxdb  104 May 15 16:26 null
-rwxrwxr-x. 1 uxdb uxdb  496 May 15 11:50 start-all.sh
-rwxrwx---. 1 uxdb uxdb 1092 May 15 11:43 stop-all.sh

[uxdb@uxmydb dfs]$ vi start-all.sh
java -Xmx1024m -XX:MaxNewSize=128m -XX:MaxPermSize=256m  -ea -cp ./java/servers/dist/XtreemFS.jar:./java/lib/BabuDB.jar:./java/flease/dist/Flease.jar:./java/lib/protobuf-java-2.5.0.jar:./java/foundation/dist/Foundation.jar:./java/lib/jdmkrt.jar:./java/lib/jdmktk.jar:./java/lib/commons-codec-1.3.jar org.xtreemfs.osd.OSD etc/xos/xtreemfs/osdconfig2.properties 2>null &
```

关于dirconfig.properties        mrcconfig.properties     的启动脚本删除。
为操作方便，也修改下环境变量：

```
[uxdb@uxmydb ~]$ vi .bash_profile
export JAVA_HOME=/home/uxdb/jdk1.8.0_20
export PATH=$JAVA_HOME/bin:$PATH
export DFS_DIR=$HOME/uxdbinstall/dfs
PATH=$PATH:$HOME/bin
PATH=$PATH:$HOME/uxdbinstall/dbsql/bin
PATH=$PATH:$DFS_DIR/bin
```

export PATH

基本的配置完成后，接下来挂载osd，在192.168.1.10上操作：

```
[uxdb@uxsinodb ~]$ mount.xtreemfs 192.168.1.10/demo ~/uxdbdata/
fuse: failed to exec fusermount: Permission denied
```

解决办法：

```
[uxdb@uxsinodb bin]$ ls -l /dev/fuse
crw-rw-rw-. 1 root root 10, 229 May 16 00:39 /dev/fuse
```

看到的用户和组都是root

切换到root用户下更改组：

```
[uxdb@uxsinodb bin]$ su - root
Password: 
[root@uxsinodb ~]# chgrp fuse /dev/fuse 
[root@uxsinodb ~]# ll /dev/fuse 
crw-rw-rw-. 1 root fuse 10, 229 May 16 00:39 /dev/fuse
```

回到uxdb用户下查看到uxdb没有fuse

```
[uxdb@uxsinodb ~]$ groups
uxdb
```

切换到root用户下加入组fuse

```
[uxdb@uxsinodb ~]$ su - root
Password: 
[root@uxsinodb ~]# usermod -a -G fuse uxdb
[root@uxsinodb ~]# su - uxdb
[uxdb@uxsinodb ~]$ groups
uxdb fuse
```

可以看到加入成功。

再次执行成功了：

```
[uxdb@uxsinodb ~]$ mount.xtreemfs 192.168.1.10/demo ~/uxdbdata
```

查看挂载情况：

```
[uxdb@uxsinodb uxdbdisk]$ xtfsutil ~/uxdbdata/
Path (on volume)     /
XtreemFS file Id     cc500496-92bc-4c64-8d80-86d76404ca8f:1
XtreemFS URL         pbrpc://uxsinodb:32638/demo
Owner                uxdb
Group                uxdb
Type                 volume
Available/Used Space 28 GB / 37 MB
Num. Files/Dirs      1297 / 28
Access Control p.    POSIX (permissions & ACLs)
OSD Selection p.     1000,3002
Replica Selection p. default
Default Striping p.  STRIPING_POLICY_RAID0 / 1 / 128kB
Default Repl. p.     not set
Snapshots enabled    no
Selectable OSDs      Default-OSD (192.168.1.10:32640)
                     Default-OSD1 (192.168.1.10:32641)
                     Default-OSD2 (192.168.1.11:32640)
```

接下来初始化db：

```
[uxdb@uxsinodb ~]$ initdb -Z -W -D uxdb01
init local dfs root:/home/uxdb/uxdbinstall/local_dfs
The files belonging to this database system will be owned by user "uxdb".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory uxdb01 ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
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
```

Success. You can now start the database server using:

    uxdb -Z -B 1024 -N 100  -D uxdb01
or

```
 ux_ctl -Z -o "-Z -B 1024 -N 100" -D uxdb01 -l logfile start
```

and
start client using:

    uxsql -d uxdb

```
[uxdb@uxsinodb ~]$ ux_ctl -Z -o "-Z -B 1024 -N 100" -D uxdb01 -l logfile start
```


server starting


uxdb=# \l
                              List of databases
   Name    | Owner | Encoding |   Collate   |    Ctype    | Access privileges 
-----------+-------+----------+-------------+-------------+-------------------
 template0 | uxdb  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/uxdb          +
           |       |          |             |             | uxdb=CTc/uxdb
 template1 | uxdb  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/uxdb          +
           |       |          |             |             | uxdb=CTc/uxdb
 uxdb      | uxdb  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(3 rows)	 

说明： OSD1单独挂载了一块磁盘，磁盘挂载方法参考  linux中将硬盘挂载在某个目录下

后记： 在挂载OSD过程中，发现OSD1没有显示出来，后来想起是没有修改OSD1配置文件中相关端口，以至于和OSD的重合了，于是更改 listen.port = 32641 （区别于osdconfig.properties配置文件中的端口号）

```
http_port = 30641 
```


鸣谢 Steven Wu



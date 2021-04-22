# UXDB实例

## 增量备份

UXDB在做写入操作时，对数据文件做的任何修改信息，首先会写入WAL日志（预写日志），然后才会对数据文件做物理修改。 当数据库服务器掉电或意外者机，UXDB在启动时会首先读取WAL日志，对数据文件进行恢复。 如果我们有一个数据库的基础备份（也称为全备），再配合WAL日志，是可以将数据库恢复到任意时间点的。

### 开启WAL归挡

#### 创建归档目录

我们在创建数据目录的同时·，创建了backups,scripts,archive_wals这几个目录，如下：/home/uxdb/uxdbinstall/dbsql

data 目录是数据目录
backups目录则存放基础备份
scripts 目录存放任务脚本
archive_wals 目录用来存放归档

 归档目录也可以是挂载的NFS目录或者磁带，注意这个目录的属主为uxdb用户即可。chown -R

#### 修改wal_level参数

wal_level参数可选的值有minimal、 replica和logical，从minimal到replica再到logical级别，WAL的级别依次增高，在WAL中包含的信息也越多。 由于minimal 这一级别的WAL不包含从基本的备份和WAL日志中重建数据的足够信息，在minimal模式下无法开启archive_mode，所以开启WAL归档wal_level至少设置为replica，如下：

```shell
ALTER SYSTEM SET wal_level='replica'
ALTER SYSTEM 
```

#### 修改archive_mode参数

archive_ mode参数可选的值有on、 off和always， 默认值为off，开启归档需要修改为on，如下：

```
ALTER SYSTEM SET archive_mode='on'; 
ALTER SYSTEM 
```

重新启动数据库使参数生效。

#### 修改archive_command参数

archive  command参数的默认值是个空字符串，它的值可以是一条shell命令或者一个复杂的shell脚本。在archive_command的shell命令或脚本中可以用“%p”表示将要归档的WAL文件的包含完整路径信息的文件名，用“%f”代表不包含路径信息的WAL文件的文件名。

一个最简单的archive_command的例子是：archive_command  =’ cp % p  /home/uxdb/uxdbinstall/dbsql/archive_wals%f’。

修改wal_level和archive mode参数都需要重新启动数据库才可以生效，修改archive_command不需要重启，只需要reload即可。 但有一点需要注意，当开启了归档，应该注意archive_command设定的归档命令是否成功执行，如果归档命令未成功执行，它会周期性地重试，在此期间已有的WAL文件将不会被复用，新产生的WAL文件会不断占用ux_wal的磁盘空间，直到ux_wal所在的文件系统被占满后数据库关闭。 由于wal_level 和archive_mode参数都需要重新启动数据库才可以生效，所以在安装结束，启动数据库之前，可以先将这些参数开启， 将archive command的值设置为永远为真的值， 例如/bin/true，当需要真正开启归档时，只需要修改archive_command的值，reload即可，而不需要由于参数调整而重启数据库。

如果考虑到归档占用较多的磁盘空间，配置归档时可以将WAL压缩之后再归档， 可以用gzip、 bzip2或lz4等压缩工具进行压缩。 当前的例子中，把archive_command设置为在ux_wal 目录使用lz4压缩WAL，并将压缩后的文件归档到/home/uxdb/uxdbinstall/dbsql/archive_wals目录：执行：

```sql
ALTER SYSTEM SET archive_command='/usr/bin/lz4 -q -z %p /home/uxdb/uxdbinstall/dbsql/archive_wals%f.lz4'; 
SELECT ux_reload conf(); 
show archive_command; 
```

#### 创建基础备份

在较低的UXDB版本中， 使用ux_start_  backup和ux_stop_backup这些低级API创建基础备份，从UXDB2.1 版开始有了ux_basebackup实用程序，使得创建基础备份更便捷，ux_ba se backup用普通文件或创建tar包的方式进行基础备份，它在内部也是使用ux_start_  backup和ux_stop_backup低级命令。 如果希望用更灵活的方式创建基础备份，例如希望通过rsync、 scp等命令创建基础备份， 依然可以使用低级API的方式。 同时，使用低级API创建基础备份也是理解PIRT的关键和基础， 我们分别对这两种方式展开讨论。

##### 使用低级API创建基础备份

使用低级API创建基础备份主要有三个步骤：执行ux_start _ backup命令开始执行备份，使用命令创建数据目录的副本和执行ux stop _  backup命令结束备份。

步骤1 执行ux_start backup命令。
ux_ start _ backup的作用是创建一个基础备份的准备，这些准备工作包括：

(1) 判断WAL归档是否已经开启。
如果WAL归档没有开启，备份依然会进行，但在备份结束后会显示提醒信息：
NOTICE:  WAL  archiving  is  not  enabled;  you  must ensure  that  all  required  WAL  segments  are  cop工ed through  other  means to  complete  the  backup 
意思是说WAL归档未启用，必须确保通过其他方式复制所有必需的WAL以完成备份。 复制所有必需的WAL听上去似乎可行，但是对于一个较大的、写入频繁的生产环境数据库来说实际是不现实的，等ux_start_ backup命令结束了再去复制必需的WAL，可能那些WAL文件已经被重用了，所以务必按照提前开启归档。

(2) 强制进入全页写模式。
判断当前配置是否为全页写模式，当full_page _ writes的值为off时表示关闭了全页写模式，如果当前配置中full_page_ writes的值为off，则强制更改，full_page_ writes的值为on，进入全页写模式；

(3) 创建一个检查点。
(4) 排他基础备份的情况下还会创建backup_label文件，一个backup_label文件包含以下五项：



- START WAL LOCATION:  25 / 2B002118  (file 00000001000000250000002B) 
- CHECKPOINT  LOCATION：记录由命令创建的检查点的LSN位置；
- BACKUP METHOD：做基础备份的方法，值为ux start backup或是ux_basebackup
- 如果只是配置流复制，BACKup METHOD的值是streamed;
  BACKUP FROM：备份来源，指是从master或standby做的基础备份；
- STARTTIME ：执行ux_start_backup的时间戳；
- LABEL：在ux_start_backup中指定的标签。
- 系统管理函数ux_start_ backup的定义如下：
  ux_start_backup(label text [, fast_boolean  [ , exclusive  boolean]]) 

该函数有一个必需的参数和两个可选参数， label参数是用户定义的备份标签字符串，一般使用备份文件名加日期作为备份标签。执行ux_start_ backup会立即开始一个CHECKPOINT操作，fast参数默认值是false， 表示是否尽快开始备份。 exclusive参数决定ux_start _  backup是否开始一次排他基础备份，也就是是否允许其他并发的备份同时进行，由于排他基础备份已经被废弃，最终将被去除，所以通常设置为false。 在系统管理函数中，有一个问一is_i飞backup函数，不能想当然地认为它是用于判断当前数据库是否有一个备份在进行，它只是检查是否在执行一个排他的备份，也就是是否有exclusive参数设置为TRUE的备份，不能用它检查是否有非排他的备份在进行。

##### 步骤2 使用命令创建数据目录的副本。

使用rsync、tar、cp、scp等命令都可以创建数据目录的副本。 在创建过程中可以排除ux_wal和ux_replslot 目录、 uxmaster.opts文件、uxmaster.pid文件，这些目录和文件对恢复并没有帮助。

##### 步骤3 执行ux_stop_backup命令

在执行ux stop _  backup命令时，进行五个操作来结束备份：

- 如果在执行ux_start  backup命令时， full-page-writes 的值曾被强制修改，则恢复到执行ux-stop-backup命令之前的值。

- 写一个备份结束的XLOG记录。

- 切换WAL段文件。

- 创建一个备份历史文件，该文件包含backup_label文件的内容以及执行ux_stop _ backup的时间戳。

- 删除backup_label文件， backup_label文件对于从基本备份进行恢复是必需的，旦进行复制，原始数据库集群中就不需要了该文件了。

  

  下面以一个创建非排他基础备份的例子演示制作基础备份的过程。

  ##### 步骤1 执行ux_start _  backup开始备份，如下：
  
  ```
  SELECT ux_start_backup（base’, false,  false); 
  ux_start_backup 
  0/4000028
(1  row) 
  ```

  ##### 步骤2 创建数据目录的副本，如下：
  
  cd  /home/uxdb/uxdbinstall/dbsql/backups 
  tar  - cvf  base.tar.gz  /home/uxdb/uxdbinstall/dbsql/
data  -- exclude=/home/uxdb/uxdbinstall/dbsql/data/uxmaster.pid exclude=/home/uxdb/uxdbinstall/dbsql/data/uxmaster.opts -- exclude=/home/uxdb/uxdbinstall/dbsql/data/log/* 
  
##### 步骤3 执行ux_stop_backup结束备份
  
  ```sql
  SELECT ux_stop_backup(false) ; 
```
  
  
这样就完成了一个制作基础备份的过程。
  
  这样就完成了一个制作基础备份的过程。

#### 使用ux_basebackup创建基础备份

+ ux_basebackup命令和相关的参数在第12章已经讲解， 这里不再赘述。 下面是一个如何使用ux basebackup创建基础备份的例子：

  ```sql
  ./ux_basebackup - Ft - Pv  - Xf  - z  -ZS  -p  1922
  
  ...
  ux_basebackup:  base backup  completed 
  ```

  看到最后一行输出为base ba c kup  completed即表示备份已经完成， 查看备份文件， 如下所示：

- ```
  ll -h  /home/uxdb/uxdbinstall/dbsql/backups/ 
  ```
```

这样就完成了一个制作基础备份的过程。



```
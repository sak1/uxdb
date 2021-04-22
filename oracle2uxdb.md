# UXDB实例

## 把Oracle迁移到UXDB

### Oracle_migration工具集简介

oracle_migration工具集的功能是：迁移Oracle表结构和主键到UXDB，以及迁移Oracle数据到UXDB。数据类型转换共包含6个工具：

- init_foreign_server: 初始化插件，在UXDB中创建Oracle Foreign Server对象，为建立外部表初始化环境
- export_oracle: 连接Oracle数据库，导出包含表结构和主键的DDL，导出过程中自动做数据类型名称转换
- import_uxdb: 连接UXDB数据库，导入包含Oracle表结构和主键的DDL，生成相同表结构的外部表和同名相同结构的本地表
- sync_table：同时连接Oracle数据库和UXDB数据库，同步表数据，同步前自动同步表结构变化和主键变化
- add_table：同时连接Oracle数据库和UXDB数据库，自动同步UXDB中已有schema范围内Oracle新增表DDL
- analyse_result：同时连接Oracle数据库和UXDB数据库，比较两边同名表记录数目，打印出记录数目不一致的地方

### Oracle_migration工具集安装

​    工具集通过Oracle Client Library远程访问Oracle，然后抽取数据到UXDB。所以先要安装好Oracle Client Library软件包和UXDB Server软件。这里做简要说明（关键是要配置好环境变量） 。

#### 安装Oracle Client Library

- 下载oracle-instantclient11.2-basic-11.2.0.3.0-1.x86_64.rpm
- root# rpm -ivh oracle-instantclient11.2-basic-11.2.0.3.0-1.x86_64.rpm
- 检查/usr/lib/oracle/11.2/client64/lib路径下是否存在libclntsh.so.11.1
- 将/usr/lib/oracle/11.2/client64/lib路径写入系统配置，找到目录/etc/ld.so.conf.d/，创建文件oraclient.conf，   打开oraclient.conf，把lib路径写入此文件并保存
- 写入配置文件后，运行sudo ldconfig使得配置生效

#### 安装UXDB Server

- 选择uxdb server安装包，麒麟机一般是uxdb-server-linux7-x64，如果是CentOS6.7则需要安装uxdb-server-linux6-x64
- 进入安装包后，直接运行./install.sh可以按照提示自动安装UXDB，关于UXDB的安装配置可以参阅UXDB的用户操作手册
- 假设安装根目录是/home/uxdb/uxdbinstall，那么在/home/uxdb/uxdbinstall/dbsql/bin下面可以找到UXDB的各种工具集
- 把/home/uxdb/uxdbinstall/dbsql/bin写入PATH环境变量	

```
vi ~/.bashrc
在bashrc中最后一行写入：
export PATH=/usr/lib/oracle/11.2/client64/lib:$PATH
```

- 写入环境变量后，source ~/.bashrc确保对本Shell生效或者重新打开一个Shell就可以在新Shell生效
- 在Shell中直接敲ux_config可以看到打印出一些安装配置信息，证明UXDB Server安装成功

#### 安装ux_migration工具集

- 解压安装包ux_migration_v2.0.tar.gz得到目录ux_migration_v2.0
- 在目录ux_migration_v2.0下面运行./install.sh
- 工具和相关部分会被复制到UXDB Server安装目录下相应位置
- 安装过程中，如果出现打印ux_config命令不存在的情况，请确认UXDB Server已经安装并且PATH环境变量已经配置
- 假设UXDB Server的安装根目录为/home/uxdb/uxdbinstall
- 那么在/home/uxdb/uxdbinstall/dbsql/bin/ux_migration下可以看见那6个工具
- 可以把/home/uxdb/uxdbinstall/dbsql/bin/ux_migration写入PATH环境变量

### Oracle_migration工具集使用

初始化一个UXDB数据库，启动数据库，具体方法参见UXDB用户操作手册。

工具集使用需要一些参数，为说明方便，假设连接信息如下：

| Oracle连接信息                                               | UXDB连接信息                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 主机 IP 和port: 192.168.138.131:1521<br/>SID: fdw_test<br />用户名称: ora_user1<br/>用户密码: ora123456<br/>业务用户名称：table_owner1 | 主机 IP 和port:192.168.138.132:5432<br/>用户名称: ux_user1<br/>用户密码: ux123456<br/>数据库名称：uxdb |

绍工具常用的参数和用法。更多使用方法可以参见帮助说明。例如：运行export_oracle -?可以看见一些使用列子和参数说明。

#### 导出DDL

```
 ./export_oracle -s 192.168.138.1:1521/fdw_test -u ora_user1 -p ora123456 -l table_owner1 > table_owner1.ddl
```

导出DDL时注意字符集的选择，果不指定字符集，默认为utf8。字符集通常要与操作系统字符集一致。

#### 初始化foreign server

首先在ux_migration目录下可以看见一个配置文件conn.config，它有六个配置项，修改这六个配置项如下：

```
oracle_server_uri=//192.168.138.131:1521/fdw_test
oracle_server_user=ora_user1
uxdb_server_host=127.0.0.1
uxdb_server_port=5432
uxdb_server_dbname=uxdb
uxdb_log_user=uxdb
```

运行init_foreign_server

```
./init_foreign_server -f conn.config -p ux123456 -r ora123456
```

init_foreign_server的参数包含UXDB连接信息和Oracle连接密码 ，此时并没有连接Oracle，只是将Oracle连接信息写人UXDB的foreign server

#### 导入DDL

```
./import_uxdb -f conn.config -p ux123456 -i table_owner1.ddl
```

导入DDL前一定要先初始化foreign server，因为导入时需要生成Oracle外部表，这些外部表依赖于foreign server

导入DDL时如果遇到同名schema，会先删除这个schema以及它包含的所有表

导入成功后，在UXDB中生成同名schema，同时生成同名表和相同表结构的外部表
列如Oracle中有张表test_table(id int,name char(20),primary key(id))
那么UXDB中生成同名本地表test_table(id int, name char(20),primary key(id))
同时生成外部表test_table_uxmigrationftbl(id int,name char(20))

#### 同步数据

```shell
./sync_table -f conn.config -p ux123456 -r ora123456 -l table_owner1 >not_sync.txt
```


同步schema为table_owner1的所有表的数据，并把同步失败的表记录到not_sync.txt文件

```
./sync_table -f conn.config -p ux123456 -r ora123456 -t table_owner1.test_table 
```

同步单表table_owner1.test_table的数据

#### 新增表DDL同步

```
./add_table -f conn.config -p ux123456 -r ora123456 -l table_owner1
```

检查schema为table_owner1的所有新增表，如果有新增那么就同步过来

```
./add_table -f conn.config -p ux123456 -r ora123456 -t table_owner1.test_table
```

检查是否有新增表table_owner1.test_table，如果有就同步过来

### 常见问题处理

#### import_uxdb导入DDL时报错

##### 遇到非法字符

这个错误常见于导出DDL文件在不同的操作系统中复制或者移动。导出DDL前先要弄清目标操作系统使用的字符集，然后在使用export_oracle导出时指定相同的字符集

##### 遇到非法数据类型

目前工具集支持绝大多数Oracle原始数据类型，不支持用户自定义数据类型。可以在导出的DDL文件中搜索[unresolved]字样，可以看见导出时没有解析成功的类型。

#### sync_table同步表数据时报错

sync_table同步数据时可能出现错误，如果是某表的问题，程序不会中断，继续同步其它表。

- 使用重定向记录下没有成功同步的表名

  ```
  ./sync_table -f conn.config -p ux123456 -r ora123456 -l table_owner1 >not_synchronized_table.txt
  ```

  not_synchronized_table.txt中记录了表名，同时简单记录了发生错误的原因。

- 如果是表结构同步失败，使用export_oracle单独导出这张表定义，与当前UXDB中DDL定义进行比较

  ```
  ./export_oracle -s192.168.138.132:1521/fdw_test -u ora_user1 
   -p ora123456 -t table_owner1.table_name
  ```

- 如果是数据同步失败，尝试使用sync_table 单独同步这张表

  ```
  ./sync_table -f conn.config -p ux123456 -r ora123456 -t table_owner1.test_table
  ```

#### sync_table选择性同步表

- 同步单张表

  ```
  ./sync_table -f conn.config -p ux123456 -r ora123456 -t table_owner1.test_table
  ```

  同步table_owner1.test_table以后（不包含，升序）的表

  ```
  ./sync_table -f conn.config -p ux123456 -r ora123456 -b table_owner1.test_table
  ```

  不同步某些表

  ```
  ./sync_table -f conn.config -p ux123456 -r ora123456 -x blacklist.txt
  ```

  其中blacklist.txt中表名字以schema_name.table_name的形式给出

- 只同步某些表

  ```
  ./sync_table -f conn.config -p ux123456 -r ora123456 -w whitelist.txt
  ```

  其中whitelist.txt中表名字以schema_name.table_name的形式给出
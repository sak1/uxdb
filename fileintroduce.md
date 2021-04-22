## UXDB实例

## 认识下主程序与数据文件

数据库主要分为程序与数据文件两个部分，以下是程序与数据库的主要内容，介绍一下

### 主要程序文件介绍

以下为/home/uxdb/uxdbinstall/dbsql/bin下程序文件，可以以命令+参数执行，具体功能介绍如下

- clusterdb	重新聚簇数据库表
- createdb	创建数据库
- createuser	创建用户
- dropdb	删除数据库
- dropuser	删除用户
- ecux	嵌入式SQL C程序预处理器
- logfile	端口监听日志
- oid2name	在数据目录中解决OID和文件节点，测试文件结构
- pgsql2shp  SQL转为SHP
- raster2pgsql  将 GDAL 支持的raster格式加载到适合加载到uxgis raster
- reindexdb	重建过引
- shp2pgsql   SHP转为SQL
- uxbench	压测工具
- uxdb 启动单独服务接受连接
- ux_dumpall	清理数据库
- ux_isready	检查数据库连接状态
- uxmaster->uxdb uxdb的别名
- ux_receivewal	从库中流式提前写日志
- ux_recvlogical	控制数据库逻辑编码流
- ux_resetwal 清理不需要的WAL文件
- ux_restore	从ux_dump建的文档中恢复数据库
- ux_rman  备份软件
- ux_standby  创建“热备用” 数据库服务器
- ux_waldump	清理WAL日志
- vacuumdb  收集垃圾并分析一个数据库
- vacuumlo	清理大对象
- initdb	创建一个新的 UXDB 数据库集群
- removedb 	删除UXDB数据库集群
- ux_archivecleanup	清理 UXDBWAL 归档文件
- ux_basebackup	获得 UXDB 集群的基础备份
- ux_config	获取已安装的 UXDB 的信息
- ux_controldata	显示一个 UXDB 数据库集群的控制信息
- ux_ctl	初始化、启动、停止或控制一个 UXDB 服务器
- ux_diagnose	分析和重置 UXDB 数据库集群的预写式日志
- ux_dump	把UXDB数据库抽取为一个脚本文件或其他归档文件
- ux_resetxlog	 清理预写日志并且可以选择地重置其它一些存储在 ux_control 文件中的控制信息
- ux_rewind	把一个 UXDB 数据目录与另一个从它复制出来的数据目录同步
- ux_set_user_priority	重置用户优先级	
- ux_test_fsync	为 UXDB 判断最快的 wal_sync_method
- ux_test_timing	度量计时开销
- ux_upgrade	升级 UXDB 服务器实例
- ux_xlogdump	从xlog中dump出一些易读的底层信息
- ux_bench	基准测试
- uxevent	事件触发器
- uxsql	UXDB 的交互式终端

### 数据文件介绍

数据目录例如：/home/uxdb/uxdbinstall/dbsql/data

| 目录/文件名        | 分类 | 描述                                                         |
| ------------------ | ---- | ------------------------------------------------------------ |
| base               | 目录 | 包含每个数据库目录，数据库目录以数据库的OID编号命名          |
| global             | 目录 | 包含整个集共享的全局表,比如ux_database                       |
| ux_commit_ts       | 目录 | 含交易提交时间戳数据的子目录                                 |
| ux_dynshmem        | 目录 | 包含动态共享内存子系统使用的文件的子目录                     |
| ux_hba.conf        | 文件 | 基于主机的访问控制文件,保存对客户端认证方式的设置信息        |
| ux_ident.conf      | 文件 | 用户名映射文件,定义了操作系统用户名和UXDB用户名之间的对应关系,这些对应关系会被ux_hba.conf用到 |
| ux_logical         | 目录 | 包含逻辑解码的状态数据的子目录                               |
| ux_multixact       | 目录 | 包含多重事务状态数据的子目录(用于共享的行锁)                 |
| ux_notify          | 目录 | 包含LISTEN/NOTIFY状态数据的子目录                            |
| ux_replslot        | 目录 | 包含复制插槽数据的子目录                                     |
| ux_serial          | 目录 | 包含已提交的序列化事务数据的子目录                           |
| ux_snapshots       | 目录 | 包含导出快照的子目录                                         |
| ux_stat            | 目录 | 包含统计子系统中的永久文件的子目录                           |
| ux_stat_tmp        | 目录 | 包含统计子系统所需临时文件的子目录                           |
| ux_subtrans        | 目录 | 包含子事务状态数据的子目录                                   |
| ux_tblspc          | 目录 | 包含用于预备事务的状态文件的子目录                           |
| ux_twophase        | 目录 | 包含用于预备事务的状态文件的子目录                           |
| UX_VERSION         | 文件 | 一个包含UXDB主版本号的文件                                   |
| ux_wal             | 目录 | 包含WAL(预写日志)文件的子目录                                |
| ux_xact            | 目录 | 包含交易提交事务的子目录                                     |
| uxsinodb.auto.conf | 文件 | 用于存储由ALTER SYSTEM设置的配置参数的文件                   |
| uxsinodb.conf      | 文件 | 主要配置文件，除基于主机的访问控制和用户名映射之外的其他用户可设置参数都保存在这个文件中 |


# UXDB实例

## uxgis

使用Uxgis之前要安装依赖库文件，依赖库包包括：json-c、gdal、geos、xml2、proj、CGAL（被SFCGAL依赖）、SFCGAL。

### 依赖库安装

依赖库文件版本与系统环境关系密切，而且依赖文件数量繁多，如果全部通过rpm命令安装的话耗时耗力且不容易成功，故建议使用yum安装，对于源上找不到的库，通过下载rpm包手动安装。对于CentOS  64 位7.4系统用户而言，可使用工具包中提供的yum源安装；对于源中找不到的库，可通过提供的rpm包或者通过网址 https://pkgs.org/ 下载安装。

UXGIS工具包	http://www.uxsino.com/uploads/soft/download/uxdb_GIS_2.4.4_tools.rar 

#### 1.使用工具包中的源。

使用工具包“Uxgis-2.4.4依赖库\yum_repo“目录中提供的源替换本地原有的yum源，放到/etc/yum.repos.d目录下。替换之前注意做好原有源的备份。

#### 2.关闭源的gpgcheck检查。

为了避免gpgcheck检查失败，在进行yum安装之前关闭所有源的gpgcheck。使用root用户
登录，执行如下操作：

```
cd /etc/yum.repos.d 
sed -i 's/gpgcheck=1/gpgcheck=0/g' *
```

#### 3.通过yum安装以下依赖包。

```
yum install json-c
yum install gdal
yum install geos
yum install xml2
yum install proj
```

#### 4.通过rpm包安装SFCGAL扩展。

root权限登录，使用 ”Uxgis-2.4.4依赖库\rpm_pkgs” 目录下rpm安装包，按如下步骤安装：

```
rpm -ivh boost-serialization-1.53.0-27.el7.x86_64.rpm
rpm -ivh mesa-libGLU-9.0.0-4.el7.x86_64.rpm
rpm -ivh CGAL-4.7-1.rhel7.x86_64.rpm
rpm -ivh SFCGAL-libs-1.3.1-1.rhel7.x86_64.rpm
rpm -ivh SFCGAL-1.3.1-1.rhel7.x86_64.rpm
```

bz2解压，安装

#### 5.验证uxgis扩展是否安装完全。

创建并启动数据库，创建如下扩展，确认uxgis扩展环境已完全搭建。

```
CREATE EXTENSION uxgis;
CREATE EXTENSION fuzzystrmatch;
CREATE EXTENSION uxgis_sfcgal;
CREATE EXTENSION uxgis_topology;
CREATE EXTENSION uxgis_tiger_geocoder; 
CREATE EXTENSION address_standardizer; 
CREATE EXTENSION address_standardizer_data_us; 
```



### 用法示例

#### 1.准备空间数据库相关的.shp文件放到demo文件夹中备用。

.shp 文件中存的是图形的信息。
.dbf 文件中存的是属性的信息。
.prj 文件中存的是坐标的信息。
.sbn和.sbx 文件中存的空间索引。
在此以导入.shp文件为例。

注意
要使用uxgis的.shp文件转换功能，必须要安装uxgis安装包（可找优炫技术人员获取），否则使用过程中会报找不到libwgeom-2.4.so.0库错误，建议将安装包安装到默认路径下，因为uxgis编译安装是在默认路径下进行的。

#### 2.Shape数据转换为sql文件。

进入uxdbinstall/dbsql/bin目录下，创建demo文件夹，使用如下命令进行转换：

```
shp2pgsql -s 3857 -c -W "GBK" /home/uxdb/Desktop/demo/hyd1_4l.shp >demo/hyd1_4l.sql
```

参数说明:
• -s 代表指定数据的SRID
• -c 代表数据将新建一个表
• -d 删除旧的表，重新建表并插入数据
• -a 向现有表中追加数据
• -p 仅创建表结构，不添加数据

• -W Shape文件中属性的字符集
有时Shape数据中的字符集是其他，就可能报“Unable  to  convert  data  value  to  UTF-8(iconv reports "无效或不完整的多字节字符或宽字符").
 Current encoding is "UTF-8". Try
"LATIN1" (Western European)”错误，这时候指定正确的字符集即可解决问题。

#### 3.建立空间数据库，并创建uxgis相关扩展。

a. 进入uxdbinstall/dbsql/bin目录下，创建数据库。

```
initdb -W -D testdb
```

b.修改配置文件testdb/uxsinodb.conf，使其能进行uxgis扩展。

去掉shared_preload_libraries的注释，进行修改。

```
shared_preload_libraries = 'postgres_adaptor'           # (change requires restart)
```

设置adaptor_pg_ux_mode_switch。

```
adaptor_pg_ux_mode_switch = on
```

c. 使用uxdb命令启动数据库，并通过uxsql连接。

```
uxdb -D testdb
```

d.创建uxgis扩展。

```
CREATE ROLE  gisdb;
CREATE DATABASE shp2pgsqldemo WITH OWNER=gisdb;
\c shp2pgsqldemo;
CREATE EXTENSION uxgis;
CREATE EXTENSION fuzzystrmatch;
CREATE EXTENSION uxgis_sfcgal;
CREATE EXTENSION uxgis_topology;
CREATE EXTENSION uxgis_tiger_geocoder; 
CREATE EXTENSION address_standardizer; 
CREATE EXTENSION address_standardizer_data_us; 
```



#### 4.使用uxsql向数据库导入使用Shape数据生成的.sql文件。

```
uxsql -d shp2pgsqldemo -f demo/hyd1_4l.sql -W
```


导入成功，结果展示：

#### 5.验证数据。

```
\d
SELECT COUNT(*) FROM hyd1_4l;
SELECT COUNT(*) FROM spatial_ref_sys;
SELECT * FROM us_lex;
```


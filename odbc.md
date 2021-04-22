# UXDB实例

## Windows下ODBC的安装与使用

ODBC（Open Database Connectivity）开放数据库连接，基于Windows环境的一种数据库访问接口标准。UXDB支持ODBC，程序员可通过在windows下安装odbc，实现与UXDB数据库的连接。

### 安装VC++ runtime

​      https://support.microsoft.com/en-us/topic/the-latest-supported-visual-c-downloads-2647da03-1eea-4433-9aff-95f26a218cc0    

### 安装ODBC

​      下载uxdb-odbc-2.1.msi到本地，双击安装

### 配置DSN

2.安装成功后，进入C:\Windows\System32\路径下，双击odbcad32.exe。
3.点击添加-->在"用户DSN"选项卡下，选择UxsinoDB Unicode；

- 修改Database为要连接的数据库名，如uxdb；  

- 修改Server为要连接服务端IP，如192.168.138.131；

- 修改Port为要连接的集群port，如5432；

- 修改User Name为要连接的数据库用户，如uxdb；

- 修改Password为要连接的数据库的密码，如123456；

- 其他默认。

  点击Test，提示连接成功，说明数据库连接成功，点击Save保存。

### 配置项目

#### 右击项目-> 属性-> 配置属性->C/C++ -> 

常规->附加包含目录
$PATH(实际解压路径)\odbcTestProGTest_windows
\odbcTestProGTest\include
$PATH(实际解压路径)\odbcTestProGTest_windows
\odbcTestProGTest\lib\win32

#### 右击项目-> 属性-> 配置属性-> 链接器-> 常规->附加库目录

$PATH(实际解压路径)\odbcTestProGTest_windows
\odbcTestProGTest\lib\Win32

#### 右击项目->属性->配置属性->链接器->输入->附加依赖项

$PATH(实际解压路径)\odbcTestProGTest_windows
\odbcTestProGTest\lib\Win32\gtestd.lib
$PATH(实际解压路径)\odbcTestProGTest_windows
\odbcTestProGTest\lib
\Win32\gtest_maind.lib
$PATH(实际解压路径)\odbcTestProGTest_windows
\odbcTestProGTest\lib\Win32\gtest.lib

#### 右击项目->添加->现有项，添加odbc_config.txt（在项目目录中：PATH

\odbcTestProGTest_windows\odbcTestProGTest\odbcTestPro\odbc_config.txt）修改odbc_config.txt的内容（根据实际数据库信息修改），如：

```
#数据源名称
g_dbdsn=UXsinoDB35W
#连接库名
db_name=mydb 
#连接数据库ip地址
host=192.168.138.131
#用户名
g_dbuser=uxdb
#密码
g_dbpwd=mypassword
#模式名
schema_name=public
```



#### 右击项目->属性->点击右上角配置管理器：

活动解决方案配置选择Debug；活动解决方案平台选择Win32；保存退出属性配置。

#### 右击项目->生成

Ctrl+F5执行用例




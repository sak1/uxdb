# UXDB实例

## 加载license
软件安装后，软件启动时系统会提示：

WARNING:  can not access license file (/home/uxdb/uxdbinstall/license/uxdb.lic
Linux下License的安装

###  1.进入UXDB安装路径下的license目录，
```
cd /home/uxdb/uxdbinstall/license
```

### 2.添加同一局域网内的其他机器的信息

```
vi IpAdd.ini
```

添加格式为机器IP:root:root的密码，例如192.168.0.124:root:000000。

> 注意：用于批量生成license，root用户和密码用来获取主板序列号，如果是本机的话不用配置，这步可省略

### 3.生成UXDB所在服务器的硬件信息

（version_type为安装版本类型；version为安装版本号）

```shell
./GetUserInfo.sh --version_type=enterprise --version=2.1.0.0
```

版本类型对应：标准版-->standard；企业版-->enterprise。
GetUserInfo.sh --help 查看该命令的详细信息。

执行后在同一目录下生成了UxdbLicense.json文件

在Xshell下按<kbd>^CTRL</kbd>+<kbd>^ALT</kbd>+<kbd>F</kbd>，新建文件传输入，把UxdbLicense.json传到本地电脑。

### 4.发送UxdbLicense.json

UxdbLicense.json文件给优炫技术人员，UXDB软件工程师会通过UxdbLicense.json生成许可证uxdb.lic

### 5.安装uxdb.lic

把uxdb.lic通过xshell拖到UXDB安装路径/home/uxdb/uxdbinstall/license目录下（与UxdbLicense.json同一个目录）

### 6.启动数据库，验证是否成功。

```
cd /home/uxdb/uxdbinstall/dbsql/bin
./ux_ctl start -D ../data
./uxsql
```


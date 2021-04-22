# UXDB实例

## 启停、登录UXDB

### 启动停止DB（LOCAL）
LOCAL本地DB不需要DFS，uxdb用户进入/home/uxdb/uxdbinstall/dbsql/bin目录下

#### 1、初始化数据集群

如已完成库的初始化，这步在此可省略。初始化过程中需要输入超级管理员uxdb的登录密码

~~~
$./initdb -W -D ../data（默认相对数据目录在/home/uxdb/uxdbinstall/dbsql/bin/下）
~~~

#### 2、启动DB server
~~~
#./uxdb -D ../data 或者./ux_ctl start -D ../data
~~~

#### 3、登录数据库

```
#./uxsql 或 ./uxsql -d uxdb
```

#### 4、列出数据库

```
uxdb=#\l 
```

#### 5、退出数据库

```
uxdb=#\q
```

#### 查看DB server状态

~~~
#./ux_ctl status -D ../data
~~~
#### 停止DB server
~~~
#./ux_ctl stop -D ../data
~~~

#### 关闭防火墙 
~~~
#systemctl stop firewalld.service
~~~



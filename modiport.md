# UXDB实例

## 修改UXDB端口

> 端口用于数据库管理与开发，默认为5432，为安全起见，可修改端口。

### 1 停止UXDB服务

~~~
#./ux_ctl stop -D ../data
~~~

### 2 修改配置
~~~
vim /home/uxdb/uxdbinstall/dbsql/data/uxsinodb.conf
#约第63行
#port = 5432     
#修改为
port = 5433
~~~

### 3 重启UXDB服务
~~~
#./ux_ctl start -D ../data
~~~

### 4 验证
>使用uxdbadmin或[第三方管理工具dbeaver连接](linkdbeaver.md)，验证。
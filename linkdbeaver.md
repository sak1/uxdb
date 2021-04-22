# UXDB实例

## dbeaver管理UXDB

如果我们使用[dbeaver](https://dbeaver.io/download/)来管理UXDB，我们需要做服务端做点工作。

### 配置UXDB参数

进入数据目录，如/home/uxdb/uxdbinstall/dbsql/data

```
vim uxsinodb.conf
#修改第151行为
shared_preload_libraries = 'postgres_adaptor'
查看 669行
adaptor_pg_ux_mode_switch = on   
#如为off，就改为on，然后保存退出
```

### 重启数据库

```shell
./ux_ctl restart -D ../data
```

### 关闭防火墙

关闭防火墙或把端口添加到防火墙中

```sh
systemctl stop firewalld.service
```

如需彻底关闭防火墙，执行

```sh
systemctl disable firewalld.service
```

### 打开dbeaver

  创新新连接，选择postgresql
  在“常规”选项卡处输入：
  server
  主机：192.168.138.132
  数据库：UXDB
  认证
  用户名：uxdb 
  密码：mypassword

点击==kdb测试链接kdb==

successed!

  
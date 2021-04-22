# UXDB实例

## 禁止其他主机访问UXDB

### 修改安全配置文件

进入数据目录，如/home/uxdb/uxdbinstall/dbsql/data

```
vim ux_hba.conf

IPv4 local connections:
host    all             all             0.0.0.0/0               reject
```

这行最后一个字符改成“reject”

### 重启服务

```
systemctl restart uxdb
```

看看，使用客户端连接数据是不是连不上了？

### 重新开启

需要重连时，再把原来的改回来哈，比如把 reject 改成md5 或trust，再重启服务，就可以了。
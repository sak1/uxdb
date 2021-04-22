# UXDB实例

## 安全授权控制

要求客户端提供受信任的证书，把你信任的根证书颁发机构（CA）的证书放置在数据目录文件中。提高安全级别；

```shell
vi uxsinodb.conf
#ssl_ca_file =‘’

vi ux_hba.conf
host	replication	all	::1/128	md5
clientcert=verify-full
```


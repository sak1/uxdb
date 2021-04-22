# UXDB实例

## 开机自启动

1 下载[uxsino](data/uxsino)文件，编辑29、32行

```sh
prefix=/home/uxdb/uxdbinstall/dbsql        
#uxdb服务目录
UXDATA="/home/uxdb/uxdbinstall/dbsql/data"		
#设置数据所在目录
```

2 通过XShell拖入或xshell FTP，把uxdb文件复制到服务器 /etc/rc.d/init.d 目录

```shell
cd /etc/rc.d/init.d
chmod a+x uxsinodb
```

3 建个日志文件

```shell
touch /home/uxdb/uxdbinstall/dbsql/data/serverlog   
chmod +x /home/uxdb/uxdbinstall/dbsql/data/serverlog
```

可能出现错误如下：Failed to get properties: Access denied 

纠错办法：

```shell
systemctl daemon-reexec
#守护程序重新加载uxsino
```

4 启动服务

```
service uxsinodb start
```

5 设置uxdb服务开机自启动

```
chkconfig --add uxsinodb
```

可能出现错误如下：service uxdb does not support chkconfig

请把 #chkconfig: - 85 15 加入到uxdb文件首行

6 测试一下

```shell
shutdown -r now
#重启
su uxdb
cd /home/uxdb/uxdbinstall/dbsql/bin
./ux_ctl status -D ../data 
#或./uxsql登录数据库
```






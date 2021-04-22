# UXDB实例

## 文件系统级别的备份

停掉运行中的数据库，并将数据目录包括表空间使用cp、 tar、 nc等命令创建一份副本，保存在合适的地方即可。

### 方法一：复制

```
./ux_ctl stop -D ../data
cp -a /home/uxdb/uxdbinstall/dbsql/data /backup/ 
```

### 方法二：打包

```
cd /backup
tar zcvf data.tar.gz /home/uxdb/uxdbinstall/dbsql/data
```

### 方法三：网络定时文件传输

NetCat是一个简单、可靠的网络工具，可通过TCP或UDP协议传输读写数据。

安装nc

```
yum install -y nc
```

【主机】编辑一个文件 client_nc.sh

```
#!/bin/bash
NC=/bin/nc
TAR=/bin/tar
BACKUP_DIR=/home/uxdb/uxdbinstall/dbsql/data   #backup source dir
PORT=1234
SERVER_IP=192.168.138.132      #backup target server ip
$TAR -zvcf - $BACKUP_DIR | $NC $SERVER_IP $PORT
```

给执行权限改，加到定时任务

```
$chmod +x clint_nc.sh
$crontab -e
#m h  dom mon dow   command
1 1 * * * /client_nc.sh
wq保存退出
$crontab -l
```

【备机】编辑一个文件  server_nc.sh

```
#!/bin/bash
NC=/bin/nc
TIMETAMP=`date +%Y%m%d%H%M%S`    
PORT=1234
$NC -l $PORT > data.$TIMETAMP.tgz
```

给执行权限改，加到定时任务

```
$ chmod +x server_nc.sh
$ crontab -e  
#m h  dom mon dow   command
0 1 * * * /server_nc.sh
wq保存退出
$ crontab -l
```

注意时钟同步

注意，适当关闭防火墙  

```
systemctl stop firewalld
```

可以即时测一下：备份机执行 ./server_nc.sh ,主机执行 ./client_nc.sh，就可以看到执行动作了，执行结束，备机上就有data.20210415200355.tgz文件了
# UXDB实例

## linux下安装、初始化uxdb

> 以安装在CentOS7.4下安装uxdb-ent-cen7x-x86_64-2.1.0.2.tar.gz为例

### CentOS下安装JDK

~~~shell
#yum search jdk
#yum install java-1.8.0-openjdk.x86_64
~~~

> 如出现Yum Command Fails with “Another app is currently holding the yum lock” in CentOS/ RHEL 7错误
~~~shell
# kill -9 13023  /* 13023 为上面的pid值: 13023
~~~
~~~shell
#java -version
OpenJDK Runtime Environment (build 1.8.0_282-b08)
OpenJDK 64-Bit Server VM (build 25.282-b08, mixed mode)
~~~

### 修改uxdb用户权限
~~~shell
#chmod 600 /etc/sudoers
#vim /etc/sudoers
root    ALL=(ALL)       ALL   后面增加一行  
uxdb    ALL=(ALL)       ALL
~~~

### 上传本地UXDB安装包
如在本地选择  uxdb-ent-cen7x-x86_64-2.1.0.2.tar.gz，该软件可在[官网](www.uxsino.com/uxdb/login.php)下载
使用[Xshell5]（https://www.netsarang.com/en/xshell/） 上传文件到CentOS下 /home/uxdb 
### 建安装目录
~~~sh
#cd /home/uxdb
#mkdir uxdbinstall
~~~
### 安装 uxdb
~~~bash
#cd /home/uxdb/
#tar xvf uxdb-ent-cen7x-x86_64-2.1.0.2.tar.gz
#cd /home/uxdb/uxdb-server-linux7-2.1.0.2-EE
#./install.sh
~~~
Please input uxdb user[uxdb]
单机环境下，对要求输入的5个？，分别输入 n，y，n，n，n，即只安装dbsql

如出现错误，请检查上面的操作是否有误。
Install UXDB success!


### 数据初始化
~~~bash
#su uxdb  /*切换用户*/
#cd /home/uxdb/uxdbinstall/dbsql/  /*切换到本目录*/
#mkdir data	  /*建数据所在目录data.data为自定义，也可以是其它名称*/
#cd /home/uxdb/uxdbinstall/dbsql/bin/  /*切换到本目录*/
#./initdb -D ../data -W  	/*初始化  -D 用于指向数据所在目录， -W 用于给超级管理员分配密码*/
Enter new superuser password:   /*输入个密码，密码不建议与uxdb用户重复，*/
~~~

Success
# UXDB实例

## uxdbadmin的安装、启动与卸载

### 安装

Xshell 登录服务器，把uxdbadmin-2.1.0.0.tar.gz拖到/home/uxdb

#### 解压安装包

~~~bash
#tar -xvf uxdbadmin-2.1.0.0.tar.gz
~~~
#### 进入解压目录，执行安装脚本：

~~~bash
#cd uxdbadmin-2.1.0.0
#./install.sh
~~~
#### 输入安装路径（默认安装路径为/home/uxdb/uxdbinstall/uxdbAdmin），回车表示默认安装

~~~bash
please input uxdbAdmin install dirctory[/home/uxdb/uxdbinstall/uxdbAdmin]:y
~~~

## 启动
进入UXDBAdmin安装路径,执行启动脚本

~~~bash
#cd /home/uxdb/uxdbinstall/uxdbAdmin/web
#vim config_local.py

修改其中的 127.0.0.1为0.0.0.0 或服务器IP如 10.1.1.108，:wq保存退出

#./start.sh
如不启动或出现错误提示
#./start.sh p
~~~

## 卸载
进入UXDBAdmin安装路径,执行卸载脚本：

~~~bash
#cd /home/uxdb/uxdbinstall/uxdbAdmin
#./uninstall.sh
~~~
确认是否卸载，回车或输入Y/y确认卸载，输入N/n终止卸载：

~~~bash
Are you sure you want to uninstall uxdbAdmin?[Y/N]:y
~~~

Successed！
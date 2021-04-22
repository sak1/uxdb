# UXDB实例

### UXDB服务重启失败怎么办？

查一下服务是否已经启动

```shell
./ux_ctl status -D ../data
```

如果结果ux_ctl: server is running (PID: 1360)，说明是已经启动。

如果不是

```shell
./ux_ctl start -D ../data
```

如果启动不起来，删除的data目录下面的uxmaster.pid文件，重启服务；如果windows下的uxdb服务启动不起来，在uxdb服务处，右键进入属性 -选择 登录 选项卡-勾选本地系统账户，再启动服务。

注意：对数据要做好备份，数据目录的任何操作都不建议轻易操作，尤其是重要数据
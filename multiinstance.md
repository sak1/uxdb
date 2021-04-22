# UXDB实例

## 单机搭建多个UXDB实例

如建两个实例：

### 初始化两个数据目录

这样可以同时启动这两个实例了，启动方法如下：
在uxdb用户下的.bash_profile文件中添加如下内容:

```
alias uxstart='ux_ctl -D $UXDATA start'
alias uxstop='ux_ctl kill INT `head -1 $UXDATA/uxmaster.pid`'
```

### 启动

启动实例1

```
export UXDATA=/data/UXDATA1
uxstart
```

启动实例2

```
export UXDATA=/data/UXDATA2
uxstart
```

### 登录

登录实例1

```
./uxsql -p 5433
```

登录实例2

```
./uxsql -p 5434
```


# UXDB实例

## 页校验功能(可开关的页校验功能)

ux_checksums  检查写入磁盘数据的一致性，以前该操作只允许在 initdb的阶段来执行

```shell
ux_checksums -e -c -D ../data
```

-e 是开启，-c 是检查 -D 是目录

这个功能在v2.1.1.2以上才有哈


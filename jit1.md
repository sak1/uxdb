# UXDB实例

## JIT编译能干什么

JIT实现支持对表达式计算以及元组拆解的加速。增强长时运行CPU密集查询。

### 使用前

```
EXPLAIN ANALYZE	SELECT SUM(relpages) from ux_class;
```

### 使用后

```
Set jit_above_cost =10;
EXPLAIN ANALYZE	SELECT SUM(relpages) from ux_class;
```


对4000万条记录库操作测试Planning time提升了千倍，Execution time 数百倍，未来更多操作有计划采用这种技术加速


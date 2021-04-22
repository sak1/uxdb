# UXDB实例

## UXDB北京某热力图模拟

用于呈现UXDB数据可视化能力

### 取北京四至：115.430651,39.448725,117.51999,41.066947

http://huiyan.baidu.com/github/tools/coord/

### UXDB下建表

```sql
create table rlt(  
  id int,    
  info text,   -- 信息  
  val float8,  -- 取值  
  ln float8,   -- 经度，la 纬度  
  la float8
); 
```

### 在bin下编辑vi rlt.sql  

```sql
insert into rlt values (random()*10000, md5(random()::text), random()*100, (random()*(117.51999-115.430651)+115.430651::int), (random()*(41.066947-39.448725)+39.448725::int));
```

### 压入数据

```shell
#./uxbench -M prepared -n -r -P 1 -f ./rlt.sql -c 50 -j 50 -t 5
```

### 取出数据

​	在uxdbadmin下找到rlt表，按F8，导出数据到csv

### 渲染数据

   https://mapv.baidu.com/editor/  

### 查看并导出结果

![如图](D:\github\sak1\uxsinodb\img\MapvOutput.png)

> 备注：也采用其它可视化工具软件如BDP，数据生成后，实时调用。


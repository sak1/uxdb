# UXDB实例

## 海量数据单机压测

### 一、测试需求
结构化数据，大约每天3000万条数据，会频繁写入新信息，不存储特征值信息，可能存在查询全量历史结果。

### 二、需求分析

1、做单表写入测试，验证3000万条数据录入所需时时长。  
2、建立索引，模拟人口信息表关多表关联查询，并验证当前数据库在此环境下可满足多少用户查询需求。

### 三、测试环境

测试软件：uxbench ，是UXDB的压力和稳定性测试工具
数据库软件 UXDB  standard v2.0.4.1

### 四、测试过程

#### 1、写入测试

##### 1 登录数据库（略）

##### 2 UXDB下建表

```
  create table cam (  
  id int,    
  camid int,  
  personid int,  
  imgurl text,
  rectime timestamp
);  
```

##### 3 编辑写入数据语句

```shell
vi cam.sql

insert into cam values(random()*10000, random()*10000, random()*10000, md5(random()::text), CURRENT_TIMESTAMP); 
```

##### 4 压入数据执行

```shell
./uxbench -M prepared -n -r -P 1 -f ./cam.sql -c 50 -j 50 -T 1800
```

##### 5 查询

```shell
select ux_size_pretty(ux_relation_size('cam'));
```

#### 2、联合查询

##### 1 建person表 ，用于联合查询 

```sql
create table person (  
  id int, 
  personid int,     
  name text
);
```

##### 2 模拟数据

​       写person.sql

```sql
insert into person values (random()*10000,random()*10000,md5(random()::text));
```

​	执行person.sql

```shell
./uxbench -M prepared -n -r -P 1 -f ./person.sql -c 50 -j 50 -T 10
```

##### 3 建索引，支持较大数据量下的查询性能

```
CREATE INDEX pindex ON cam(personid);   
```

##### 4 查询语句写入sql.sql

```shell
select count(*) as abc from cam ,person where cam.personid = person.personid and ...
```

(略，建议回归增加更多联合SQl语句);

##### 5 压测,50并发

```shell
./uxbench -M prepared -n -r -P 1 -f ./sql.sql -c 50 -j 50 -T 300
```


TPS:394

## 五、测试结论

### 入库时长：满足

TPS：17227，实测：3000万条数据耗时：29分钟，超出每天3000万录入预期；此外，产生数据量：2500MB，按此计算，保持一倍余量，年度最低硬盘容量应在2TB。

### 查询效率评估：满足

双表联合查询，其中一表数据3000万，建索引,50并发，TPS：394。可充分满足规模用户的查询应用。
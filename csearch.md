# UXDB实例

### 全文检索的实现

相比模糊检索，全文检索的优势：
1 正规化
2 根据近似度进行排名
3 可以使用索引

### 数据样例

### tmp.txt

```asciiarmor
888,A019,000000428601,0000,00000,"",0000,青岛业务支持中心,20200410,165004,22
888,A019,000000010632,0000,00001,"",0000,电信广场5楼,20200623,083204,23
888,A019,000000428609,0000,00000,"",0000,南京管线维护站,20200410,165004,24
888,A019,000000428637,0000,00000,"",0000,北海市农业办公楼,20200410,165004,25
```

~100万条

### 安装全中文检索支持

zhparser是基于Simple Chinese Word Segmentation(SCWS)中文分词库实现的一个扩展，作者 amutu，源码 https://github.com/amutu/zhparser。

```shell
create extension zhparser;
```

### 新增检索配置

```shell
CREATE TEXT SEARCH CONFIGURATION testzhcfg (PARSER = zhparser);
```

### 增加标记字典 

其中simple为一个字典模板

```sql
ALTER TEXT SEARCH CONFIGURATION testzhcfg ADD MAPPING FOR n,v,a,i,e,l WITH simple;
```

### 建表语句 

```sql
 create table anlh_tmp(
  mandt   varchar(20) default '000' not null,
  bukrs   varchar(20) default '' not null,
  anln1   varchar(50) default '' not null,
  luntn   varchar(20) default '',
  lanep   varchar(20) default '00000',
  anupd   varchar(20) default '',
  funtn   varchar(20) default '',
  anlhtxt text default '',
  zdate   varchar(32) default '00000000',
  ztime   varchar(32) default '000000',
  mykey serial primary key
);
```

### 导入数据

```sql
 \copy anlh_tmp from '/home/uxdb/tmp.txt' DELIMITER ',' csv;
```

### 建立分词库

```sql
create table anlh_ts(mykey int,anlhtxt tsvector);

insert into anlh_ts select mykey,to_tsvector('testzhcfg',anlhtxt) from anlh_tmp;	

select count(*) from anlh_tmp where anlhtxt like '%办公室%';

select count(*) from anlh_tmp t join anlh_ts s on t.mykey=s.mykey  where s.anlhtxt @@ to_tsquery('testzhcfg','办公室');
```

### 查看执行计划

```shell
explain select count(*) from anlh_tmp where anlhtxt like '%办公室%';

explain select count(*) from anlh_tmp t join anlh_ts s on t.mykey=s.mykey  where s.anlhtxt @@ to_tsquery('testzhcfg','办公室');
```

### 进行排名

```sql
SELECT anlhtxt, ts_rank_cd(to_tsvector('testzhcfg',anlhtxt),to_tsquery('testzhcfg','办公室') ) AS rank
FROM anlh_tmp
ORDER BY rank DESC
limit 5;
anlhtxt                             | rank
----------------------------------------------------------------+------
 北滘君兰机场办公室外墙光分箱-机场办公室ZHX               |  0.2
 办公室，二车间办公室                                  |  0.2
 均安鸿达服装公司办公室至鸿达服装办公室基站                |  0.2
 凤凰南路59号香洲图书馆3F办公室墙壁                      |  0.1
 市信息大厦12楼办公室                                  |  0.1
(5 rows)
```

最后，把上面的SQL引用到用户搜索页即可。
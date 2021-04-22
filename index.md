# UXDB实例

## 认识下索引

索引是一种按列对行值排序的存储结构，类似于图书的目录，可以根据目录中的页码快速找到所需的内容。

一用：如果表中某列频繁查询，那就在该列上创建索引。

四不用：1.小表不用、2.频繁大量更新操作的表不用、3.NULL值的列不用、4.频繁非查询操作的列不用。

### 创建索引

```
CREATE [ UNIQUE ] INDEX [ CONCURRENTLY ] [ [ IF NOT EXISTS ] name ] ON [ ONLY ] table_name [ USING method ]
    ( { column_name | ( expression ) } [ COLLATE collation ] [ opclass ] [ ASC | DESC ] [ NULLS { FIRST | LAST } ] [, ...] )
    [ INCLUDE ( column_name [, ...] ) ]
    [ WITH ( storage_parameter = value [, ... ] ) ]
    [ TABLESPACE tablespace_name ]
    [ WHERE predicate ]
```

### 常用参数

- unique	创建唯一索引
- concurrently	在线创建索引，不会阻塞写操作
- using method	默认为btree
- asc/desc	排序，默认升序
- nulls first/last	空值在索引中的位置，默认null放在最后

### 索引类型

btree、hash、gin、gist、sp-gist、brin、rum、bloom

#### btree 二叉树

适合处理那些按顺序存储的数据之上的等于和范围查询

操作符：<、<=、=、=、>、between ... and ...、in 

```sql
create table t_btree(id int,info text);
insert into t_btree select generate_series(1,10000),md5(random()::text);
explain analyze select * from t_btree where id = 1;
                                             QUERY PLAN                                              
-----------------------------------------------------------------------------------------------------
 Seq Scan on t_btree1  (cost=0.00..73.88 rows=26 width=36) (actual time=0.001..0.001 rows=0 loops=1)
   Filter: (id = 1)
 Planning time: 0.046 ms
 Execution time: 0.763ms
(4 rows)
```

```sql
create index i_btree_id on t_btree(id);
explain analyze select * from t_btree where id =1;
QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on t_btree1  (cost=4.36..14.68 rows=26 width=36) (actual time=0.003..0.003 rows=0 loops=1)
   Recheck Cond: (id = 1)
   ->  Bitmap Index Scan on i_btree_id  (cost=0.00..4.35 rows=26 width=0) (actual time=0.002..0.002 rows=0 loops=1
)
         Index Cond: (id = 1)
 Planning time: 0.209 ms
 Execution time: 0.025 ms
(6 rows)
```

**10000条数据，建索引相比不建索引查询提升了几十倍**

#### hash 哈希索引

处理简单的等于比较

```sql
create table t_hash(id int,info text);
insert into t_hash select generate_series(1,1000),repeat(md5(random()::text),10000);
explain analyze select * from t_hash where info = (select info from t_hash limit 1);
QUERY PLAN 
------------------------------------------------------------------------------------------------------------
 Seq Scan on t_hash  (cost=0.14..137.63 rows=1 width=3712) (actual time=0.486..210.770 rows=1 loops=1)
   Filter: (info = $0)
   Rows Removed by Filter: 999
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.00..0.14 rows=1 width=3708) (actual time=0.002..0.002 rows=1 loops=1)
           ->  Seq Scan on t_hash t_hash_1  (cost=0.00..135.00 rows=1000 width=3708) (actual time=0.001..0.001 row
s=1 loops=1)
 Planning time: 0.104 ms
 Execution time: 210.787 ms
(8 rows)
```

```sql
create index i_hash_info on t_hash using btree(info);
explain analyze select * from t_hash where info = (select info from t_hash limit 1); 
QUERY PLAN             
------------------------------------------------------------------------------------------------------------
 Index Scan using i_hash_info on t_hash  (cost=0.66..8.68 rows=1 width=3712) (actual time=3.131..3.131 rows=1 loop
s=1)
   Index Cond: (info = $0)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.00..0.14 rows=1 width=3708) (actual time=0.007..0.007 rows=1 loops=1)
           ->  Seq Scan on t_hash t_hash_1  (cost=0.00..135.00 rows=1000 width=3708) (actual time=0.006..0.006 row
s=1 loops=1)
 Planning time: 0.215 ms
 Execution time: 3.175 ms
(7 rows)
```



#### gin 倒排索引

gin是倒排索引，

适合搜索多值类型的直接检索，如数组、全文检索、TOKEN。支持相交、包含、大于、在左边、在右边等操作符号，当用户的数据比较稀疏时，如果要搜索某个VALUE的值，可以适应btree_gin支持普通btree支持的类型。

1、多值类型搜索

```sql
create table t_gin1(id int,arr int[]);

do language plpgsql $$
declare
begin
for i in 1..10000 loop
insert into t_gin1 select i,array(select random()*1000 from generate_series(1,10));
end loop;
end;
$$;

create index i_gin1_id on t_gin1 using gin(arr);
explain analyze select * from t_gin1 where arr && array[1,2];
                                               QUERY PLAN                                                  
     
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on t_gin1  (cost=12.77..45.02 rows=100 width=36) (actual time=0.091..0.139 rows=195 loops=1)
   Recheck Cond: (arr && '{1,2}'::integer[])
   Heap Blocks: exact=31
   ->  Bitmap Index Scan on i_gin1_id  (cost=0.00..12.75 rows=100 width=0) (actual time=0.084..0.084 rows=195 loop
s=1)
         Index Cond: (arr && '{1,2}'::integer[])
 Planning time: 0.102 ms
 Execution time: 0.186 ms
(7 rows)
```

2 单值稀疏检索

```
create extension btree_gin;  
create table t_gin2(id int,c1 int);
insert into t_gin2 select generate_series(1,100000),random()*10;
create index i_gin2_c1 on t_gin2 using gin(c1);
explain analyze select * from t_gin2 where c1 =1;
   QUERY PLAN                                               
          
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on t_gin2  (cost=163.54..628.95 rows=19553 width=8) (actual time=1.068..4.209 rows=19988 loops=1
)
   Recheck Cond: (c1 = 1)
   Heap Blocks: exact=221
   ->  Bitmap Index Scan on i_gin2_c1  (cost=0.00..158.65 rows=19553 width=0) (actual time=1.044..1.044 rows=19988
 loops=1)
         Index Cond: (c1 = 1)
 Planning time: 0.108 ms
 Execution time: 4.979 ms
(7 rows)
```

多列任意搜索

```
create table t_gin3(id int,c1 int,c2 int ,c3 int ,c4 int);
insert into t_gin3 select generate_series(1,100000),random()*10,random()*20,random()*30,random()*40;
create index i_gin3_all on t_gin3 using gin(c1,c2,c3,c4);
explain analyze select * from t_gin3 where c1=1 or c2=2 or c3=3 or c4 =4;
    QUERY PLAN 
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on t_gin3  (cost=64.98..263.99 rows=1985 width=20) (actual time=1.476..3.431 rows=19308 loops=1)
   Recheck Cond: ((c1 = 1) OR (c2 = 2) OR (c3 = 3) OR (c4 = 4))
   Heap Blocks: exact=159
   ->  BitmapOr  (cost=64.98..64.98 rows=2000 width=0) (actual time=1.454..1.454 rows=0 loops=1)
         ->  Bitmap Index Scan on i_gin3_all  (cost=0.00..15.75 rows=500 width=0) (actual time=0.769..0.769 rows=9
948 loops=1)
               Index Cond: (c1 = 1)
         ->  Bitmap Index Scan on i_gin3_all  (cost=0.00..15.75 rows=500 width=0) (actual time=0.361..0.361 rows=4
962 loops=1)
               Index Cond: (c2 = 2)
         ->  Bitmap Index Scan on i_gin3_all  (cost=0.00..15.75 rows=500 width=0) (actual time=0.204..0.204 rows=3
351 loops=1)
               Index Cond: (c3 = 3)
         ->  Bitmap Index Scan on i_gin3_all  (cost=0.00..15.75 rows=500 width=0) (actual time=0.119..0.119 rows=2
510 loops=1)
               Index Cond: (c4 = 4)
 Planning time: 0.136 ms
 Execution time: 4.010 ms
(14 rows)
```

#### gist 索引

gist 索引不是单独一种索引类型，而是一种架构，可以在这种架构上实现很多不同的索引策略。 

操作符：<<   、&<  、&>  、<<| 、&<|、|&>、|>>、@>、<@、~=、&&

不同的类型，支持的索引检索也各不一样。例如：
1、几何类型，支持位置搜索（包含、相交、在上下左右等），按距离排序。
2、范围类型，支持位置搜索（包含、相交、在左右等）。
3、IP类型，支持位置搜索（包含、相交、在左右等）。
4、空间类型（PostGIS），支持位置搜索（包含、相交、在上下左右等），按距离排序。
5、标量类型，支持按距离排序。

1、几何类型检索

```sql
create table t_gist (id int, pos point);
insert into t_gist select generate_series(1,100000), point(round((random()*1000)::numeric, 2), round((random()*1000)::numeric, 2));
select * from t_gist  limit 3;  
id |       pos       
----+-----------------
  1 | (30.15,321.72)
  2 | (770.89,899.74)
  3 | (592.38,194.46)
(3 rows)

create index idx_t_gist_1 on t_gist using gist (pos);
explain (analyze,verbose,timing,costs,buffers) select * from t_gist where circle '((100,100) 10)'  @> pos;  
     QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.t_gist  (cost=5.06..153.55 rows=100 width=20) (actual time=0.064..0.090 rows=36 loops=
1)
   Output: id, pos
   Recheck Cond: ('<(100,100),10>'::circle @> t_gist.pos)
   Heap Blocks: exact=32
   Buffers: shared hit=34
   ->  Bitmap Index Scan on idx_t_gist_1  (cost=0.00..5.03 rows=100 width=0) (actual time=0.058..0.058 rows=36 loo
ps=1)
         Index Cond: ('<(100,100),10>'::circle @> t_gist.pos)
         Buffers: shared hit=2
 Planning time: 0.147 ms
 Execution time: 0.131 ms
(10 rows)

explain (analyze,verbose,timing,costs,buffers) select * from t_gist where circle '((100,100) 1)' @> pos order by pos <-> '(100,100)' limit 10;
                                                              QUERY PLAN 
------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.28..31.71 rows=10 width=28) (actual time=0.045..0.045 rows=0 loops=1)
   Output: id, pos, ((pos <-> '(100,100)'::point))
   Buffers: shared hit=2
   ->  Index Scan using idx_t_gist_1 on public.t_gist  (cost=0.28..314.53 rows=100 width=28) (actual time=0.044..0
.044 rows=0 loops=1)
         Output: id, pos, (pos <-> '(100,100)'::point)
         Index Cond: ('<(100,100),1>'::circle @> t_gist.pos)
         Order By: (t_gist.pos <-> '(100,100)'::point)
         Buffers: shared hit=2
 Planning time: 0.055 ms
 Execution time: 0.062 ms
(10 rows)
```

2、标量类型排序

```sql
create extension btree_gist;  
create index idx_t_btree_2 on t_btree using gist(id);  
explain (analyze,verbose,timing,costs,buffers) select * from t_btree order by id <-> 100 limit 1;  
                                                                QUERY PLAN     
------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.15..0.18 rows=1 width=41) (actual time=0.051..0.051 rows=1 loops=1)
   Output: id, info, ((id <-> 100))
   Buffers: shared hit=3
   ->  Index Scan using idx_t_btree_2 on public.t_btree  (cost=0.15..340.15 rows=10000 width=41) (actual time=0.05
0..0.050 rows=1 loops=1)
         Output: id, info, (id <-> 100)
         Order By: (t_btree.id <-> 100)
         Buffers: shared hit=3
 Planning time: 0.087 ms
 Execution time: 0.082 ms
(9 rows)
```

#### sp-gist索引

SP-GiST类似GiST，是一个通用的索引接口，但是SP-GIST使用了空间分区的方法，使得SP-GiST可以更好的支持非平衡数据结构

范围类型搜索

```
create table t_spgist (id int, rg int4range);
insert into t_spgist select id, int4range(id, id+(random()*200)::int) from generate_series(1,100000) t(id);
select * from t_spgist  limit 3;
 id |   rg    
----+---------
  1 | [1,199)
  2 | [2,84)
  3 | [3,26)
(3 rows)
set maintenance_work_mem ='32GB';
create index idx_t_spgist_1 on t_spgist using spgist (rg);
explain (analyze,verbose,timing,costs,buffers) select * from t_spgist where rg && int4range(1,100);
   QUERY PLAN  
------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.t_spgist  (cost=4.97..146.74 rows=89 width=17) (actual time=0.034..0.040 rows=99 loops
=1)
   Output: id, rg
   Recheck Cond: (t_spgist.rg && '[1,100)'::int4range)
   Heap Blocks: exact=1
   Buffers: shared hit=3
   ->  Bitmap Index Scan on idx_t_spgist_1  (cost=0.00..4.95 rows=89 width=0) (actual time=0.028..0.028 rows=99 lo
ops=1)
         Index Cond: (t_spgist.rg && '[1,100)'::int4range)
         Buffers: shared hit=2
 Planning time: 0.191 ms
 Execution time: 0.063 ms
(10 rows)

set enable_bitmapscan=off;
explain (analyze,verbose,timing,costs,buffers) select * from t_spgist where rg && int4range(1,100);
                                                             QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Index Scan using idx_t_spgist_1 on public.t_spgist  (cost=0.28..285.84 rows=89 width=17) (actual time=0.043..0.05
2 rows=99 loops=1)
   Output: id, rg
   Index Cond: (t_spgist.rg && '[1,100)'::int4range)
   Buffers: shared hit=3
 Planning time: 0.049 ms
 Execution time: 0.079 ms
(6 rows)
```

#### brin索引

brin 索引是块级索引，有别于b-tree等索引，brin记录并不是以行号为单位记录索引明细，而是记录每个数据块或者每段连续的数据块的统计信息。因此brin索引空间占用特别的小，对数据写入、更新、删除的影响也很小。
brin属于lossly索引，当被索引列的值与物理存储相关性很强时，brin索引的效果非常的好。
例如时序数据，在时间或序列字段创建brin索引，进行等值、范围查询时效果很棒。

```
create table t_brin (id int, info text, crt_time timestamp);
insert into t_brin select generate_series(1,1000000), md5(random()::text), clock_timestamp();
select ctid,* from t_brin limit 3;
 ctid  | id |               info               |          crt_time          
-------+----+----------------------------------+----------------------------
 (0,1) |  1 | 32296ee654a18967bec6ded34c2ddd2e | 2021-04-13 15:45:12.991568
 (0,2) |  2 | cdf9425d7e525c65246b32d18f113e6a | 2021-04-13 15:45:12.991808
 (0,3) |  3 | d1aad64c45af3915cc8aad5666953353 | 2021-04-13 15:45:12.991812
(3 rows)
select correlation from ux_stats where tablename='t_brin' and attname='id';
 correlation 
-------------
           1
(1 row)
select correlation from ux_stats where tablename='t_brin' and attname='crt_time';
 correlation 
-------------
           1
(1 row)
create index idx_t_brin_1 on t_brin using brin (id) with (pages_per_range=1);
create index idx_t_brin_2 on t_brin using brin (crt_time) with (pages_per_range=1);
explain (analyze,verbose,timing,costs,buffers) select * from t_brin where id between 100 and 200;
                                                         QUERY PLAN                                    
                      
-----------------------------------------------------------
 Gather  (cost=1000.00..9586.70 rows=107 width=45) (actual time=0.728..126.638 rows=101 loops=1)
   Output: id, info, crt_time
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=2326
   ->  Parallel Seq Scan on public.t_brin  (cost=0.00..8576.00 rows=45 width=45) (actual time=0.005..16
.661 rows=34 loops=3)
         Output: id, info, crt_time
         Filter: ((t_brin.id >= 100) AND (t_brin.id <= 200))
         Rows Removed by Filter: 333300
         Buffers: shared hit=2326
         Worker 0: actual time=0.001..0.001 rows=0 loops=1
         Worker 1: actual time=0.001..0.001 rows=0 loops=1
 Planning time: 0.166 ms
 Execution time: 127.649 ms
(14 rows)

explain (analyze,verbose,timing,costs,buffers) select * from t_brin where crt_time between '2020-06-27 22:50:19.172224' and '2020-06-27 22:50:19.182224';
                                                                                     QUERY PLAN        
                                                                              
-------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..9576.10 rows=1 width=45) (actual time=23.924..23.924 rows=0 loops=1)
   Output: id, info, crt_time
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=981
   ->  Parallel Seq Scan on public.t_brin  (cost=0.00..8576.00 rows=1 width=45) (actual time=20.607..20
.607 rows=0 loops=3)
         Output: id, info, crt_time
         Filter: ((t_brin.crt_time >= '2020-06-27 22:50:19.172224'::timestamp without time zone) AND (t
_brin.crt_time <= '2020-06-27 22:50:19.182224'::timestamp without time zone))
         Rows Removed by Filter: 333333
         Buffers: shared hit=2326
         Worker 0: actual time=19.066..19.066 rows=0 loops=1
           Buffers: shared hit=675
         Worker 1: actual time=19.076..19.076 rows=0 loops=1
           Buffers: shared hit=670
 Planning time: 0.058 ms
 Execution time: 26.647 ms
(16 rows)
```

#### rum索引

rum 是一个索引插件，适合全文检索，属于gin的增强版本。
增强包括：
1、在rum索引中，存储了lexem的位置信息，所以在计算ranking时，不需要回表查询（而gin需要回表查询）。
2、rum支持phrase搜索，而gin无法支持。
3、在一个rum索引中，允许用户在posting tree中存储除ctid（行号）以外的字段value，例如时间戳。

这使得rum不仅支持gin支持的全文检索，还支持计算文本的相似度值，按相似度排序等。同时支持位置匹配，例如（速度与激情，可以采用"速度" <2> "激情" 进行匹配，而gin索引则无法做到）
位置信息如下

```
select to_tsvector('english', 'hello steven');
select to_tsvector('english', 'hello i steven');
select to_tsvector('english', 'hello i am steven');
select to_tsquery('english', 'hello <1> steven');
select to_tsquery('english', 'hello <2> steven');
select to_tsquery('english', 'hello <3> steven');
select to_tsvector('hello steven') @@ to_tsquery('english', 'hello <1> steven');
select to_tsvector('hello steven') @@ to_tsquery('english', 'hello <2> steven');
select to_tsvector('hello i steven') @@ to_tsquery('english', 'hello <2> steven');
```

例子

```
create table rum_test(c1 tsvector);
create index rumidx on rum_test using rum (c1 rum_tsvector_ops);
$ vi test.sql
insert into rum_test select to_tsvector(string_agg(c1::text,',')) from  (select (100000*random())::int from generate_series(1,100)) t(c1);

$ uxbench -m prepared -n -r -p 1 -f ./test.sql -c 50 -j 50 -t 200000
explain analyze select * from rum_test where c1 @@ to_tsquery('english','1 | 2') order by c1 <=> to_tsquery('english','1 | 2') offset 19000 limit 100;
create table test15(c1 tsvector);

insert into test15 values (to_tsvector('jiebacfg', 'hello china, i''m steven')), (to_tsvector('jiebacfg', 'hello world, i''m uxdb')), (to_tsvector('jiebacfg', 'how are you, i''m steven'));

select * from test15;
create index idx_test15 on test15 using rum(c1 rum_tsvector_ops);
select *,c1 <=> to_tsquery('hello') from test15;
explain select *,c1 <=> to_tsquery('uxdb') from test15 order by c1 <=> to_tsquery('uxdb');
```



#### bloom索引

bloom索引接口是uxdb基于bloom filter构造的一个索引接口，属于lossy索引，可以收敛结果集(排除绝对不满足条件的结果，剩余的结果里再挑选满足条件的结果)，因此需要二次check，bloom支持任意列组合的等值查询。bloom存储的是签名，签名越大，耗费的空间越多，但是排除更加精准。有利有弊。

```
create index bloomidx on tbloom using bloom (i1,i2,i3) with (length=80, col1=2, col2=2, col3=4);
```

签名长度 80 bit, 最大允许4096 bits
col1 - col32，分别指定每列的bits，默认长度2，最大允许4095 bits.
bloom提供了一种基于bloom过滤器的索引访问方法。
布隆过滤器是一种节省空间的数据结构，用于测试元素是否为集合的成员。对于索引访问方法，它允许通过签名（其大小在创建索引时确定）快速排除不匹配的元组。
当表具有许多属性并且查询测试它们的任意组合时，这种类型的索引最有用。
bloom索引适合多列任意组合查询。
例子

```
create table tbloom as
   select
     (random() * 1000000)::int as i1,
     (random() * 1000000)::int as i2,
     (random() * 1000000)::int as i3,
     (random() * 1000000)::int as i4,
     (random() * 1000000)::int as i5,
     (random() * 1000000)::int as i6
   from
  generate_series(1,10000000);
select 10000000
create index bloomidx on tbloom using bloom (i1, i2, i3, i4, i5, i6);
select ux_size_pretty(ux_relation_size('bloomidx'));
create index btreeidx on tbloom (i1, i2, i3, i4, i5, i6);
select ux_size_pretty(ux_relation_size('btreeidx'));
explain analyze select * from tbloom where i2 = 898732 and i5 = 123451;
```

#### 组合索引

**btree**

b-tree多列索引支持任意列的组合查询，最有效的查询还是包含驱动列条件的查询。

**gin**

gin多列索引支持任意列的组合查询，任意查询条件的查询效率都是一样的。(不支持排序)

**gist**

驱动列的选择性决定了需要扫描多少索引条目，与非驱动列无关（而b-tree是与非驱动列也有关的）。所以并不建议使用gist多列索引，如果一定要使用GIST多列索引，请一定要把选择性好的列作为驱动列。

#### 条件索引

查询时，强制过滤掉某些条件

```
create index i1 on t1 where c1 !=1;
```

#### 表达式索引

```
查询条件为表达式时
select * from t2 where (a||' '||b) ='aaa bbb'
create index i2 on t2 (a||' '||b);
尽量不要把表达式、函数放到查询条件中
```

### 索引使用技巧

是否使用索引和什么有关系？

1. 能否走索引，是操作符是否支持对应的索引访问方法来决定的。

2. 是否用索引是优化器决定的.

如果使用索引的成本低，可以使用索引。 

或者使用了开关，禁止全表扫，也可以使用索引。

#### Planner 配置

```
#enable_bitmapscan = on
#enable_hashagg = on
#enable_hashjoin = on
#enable_indexscan = on
#enable_material = on
#enable_mergejoin = on
#enable_nestloop = on
#enable_seqscan = on
#enable_sort = on
#enable_tidscan = on
#set session enable_seqscan= off;
```

### 使用技巧

在选择性较好的列上创建索引

同一个表避免创建过多索引

建议在关联的外键字段上创建索引

### 常见不使用索引情况

- 数据类型不匹配、操作符不匹配
- where子句进行表达式或函数操作
- like的全模糊匹配(btree)
- 数据占比
- 表分析(vacuum analyze)


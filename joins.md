# UXDB实例

## 集合

集合是指具有某种特定属性的事物的总体，有点抽象，比如：一大群人，有男有女，我们要找出20岁的人，20岁就是“特定属性”，这群20岁的人就是“集合”的一种。每一种集合代表了一种选择，只是根据业务的需要，集合之间并无高下优劣之分。相对较难的倒不是概念，而是各种join的写法，下面的七种范式代表七种集合：

### 七种范式

------

#### 1 左联接

<img src="img\i1.gif" style="zoom:40%;" />  

```sql
SELECT <select list> FROM TableA A LEFT JOIN TableB B ON A.Key= B.Key;
```



------

#### 2 右联接

<img src="img\i2.gif" style="zoom:40%;" /> 

```sql
SELECT <select list> FROM TableA A RIGHT JOIN TableB B ON A.Key B.Key;
```



------

#### 3 子集

<img src="img\i3.gif" style="zoom:40%;" />

```sql
SELECT <select list> FROM TableA A INNER JOIN TableB B ON A.Key=B.Key;
```



------

#### 4 合集

<img src="img\i4.gif" style="zoom:40%;" />

```sql
SELECT <select list> FROM TableA A FULL OUTER JOIN TableB B ON A.Key= B.Key;
```



------

#### 5 子集除外

<img src="img\i5.gif" style="zoom:40%;" />

```sql
SELECT <select list> FULL OUTER JOIN TableB B ON A.Key=B.Key WHERE A.Key IS NULL OR B.Key IS NULL;
```



------

#### 6 相对补集 A \B

<img src="img\i6.gif" style="zoom:40%;" />

```sql
SELECT <select list> FROM TableA A LEFT JOIN TableB B ON AKey=B.Key WHERE B.Key IS NULL;
```



****

#### 7 相对补集 B \ A

<img src="img\i7.gif" style="zoom:40%;" />

```sql
SELECT <select list> FROM TableA A RIGHT JOIN TableB B ON A.Key= B.Key WHERE A.Key IS NULL;
```



------

下面我们以网站和访问记录为例，演示体会下集合的写法

### 模拟数据

表A site：网站信息

```sql
create table site(id serial PRIMARY KEY,name varchar(20), alexa int,country varchar(20));   
insert into site (name,alexa,country)  values('Google',1,'USA');
insert into site (name,alexa,country)  values('JD',13,'CN');
insert into site (name,alexa,country)  values('Jiumodiary',4699,'CN');
insert into site (name,alexa,country)  values('Caixin',300,'CN');
insert into site (name,alexa,country) values('Facebook',34,'USA');
insert into site (name,alexa,country)  values('Google',1,'USA');
insert into site (name,alexa,country) values('wikipedia',100,'IND');
delete from site where id =6;
```

```sql
SELECT * FROM site;
 id |    name    | alexa | country 
----+------------+-------+---------
  1 | Google     |     1 | USA
  2 | JD         |    13 | CN
  3 | Jiumodiary |  4699 | CN
  4 | Caixin     |   300 | CN
  5 | Facebook   |    34 | USA
  7 | wikipedia  |   100 | IND
(6 rows)
```

表B log：访问记录

```sql
create table log(aid serial,site_id int,acount int,adate DATE);
insert into log (site_id,acount,adate) values(1,45,to_date('2020-5-10','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(3,100,to_date('2020-5-13','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(1,230,to_date('2020-5-14','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(2,10,to_date('2020-5-14','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(5,205,to_date('2020-5-13','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(4,13,to_date('2020-5-10','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(3,220,to_date('2020-5-15','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(6,545,to_date('2020-5-16','YYYY-M-DD'));
insert into log (site_id,acount,adate) values(5,201,to_date('2020-5-17','YYYY-M-DD'));
```

```sql
SELECT * FROM log;
 aid | site_id | acount |   adate    
-----+---------+--------+------------
   1 |       1 |     45 | 2020-01-10
   2 |       3 |    100 | 2020-01-13
   3 |       1 |    230 | 2020-01-14
   4 |       2 |     10 | 2020-01-14
   5 |       5 |    205 | 2020-01-13
   6 |       4 |     13 | 2020-01-10
   7 |       3 |    220 | 2020-01-15
   8 |       6 |    545 | 2020-01-16
   9 |       5 |    201 | 2020-01-17
(9 rows)
```

### 联合查询演示

以下是对两表联合查询，显示网站访问量的不同情况

#### **1 左联接 **  以站名为准，显示每个站的访问情况，没访问量也算上

```sql
SELECT A.name,B.acount,B.adate FROM site A LEFT JOIN log B ON A.id= B.site_id;
    name    | acount |   adate    
------------+--------+------------
 Google     |     45 | 2020-01-10
 Jiumodiary |    100 | 2020-01-13
 Google     |    230 | 2020-01-14
 JD         |     10 | 2020-01-14
 Facebook   |    205 | 2020-01-13
 Caixin     |     13 | 2020-01-10
 Jiumodiary |    220 | 2020-01-15
 Facebook   |    201 | 2020-01-17
 wikipedia  |        | 
(9 rows)
```

#### **2 右联接 **  以记录为准，显示每个访问记录，没站名也算上

```sql
SELECT A.name,B.acount,B.adate FROM site A RIGHT JOIN log B ON A.id=B.site_id;
    name    | acount |   adate    
------------+--------+------------
 Google     |     45 | 2020-01-10
 Jiumodiary |    100 | 2020-01-13
 Google     |    230 | 2020-01-14
 JD         |     10 | 2020-01-14
 Facebook   |    205 | 2020-01-13
 Caixin     |     13 | 2020-01-10
 Jiumodiary |    220 | 2020-01-15
            |    545 | 2020-01-16
 Facebook   |    201 | 2020-01-17
(9 rows)
```

#### **3 子集**  显示信息完整的（有站名+有记录）， 相当于优选，只要完整的

```sql
SELECT A.name,B.acount,B.adate FROM site A INNER JOIN log B ON A.id=B.site_id;
    name    | acount |   adate    
------------+--------+------------
 Google     |     45 | 2020-01-10
 Jiumodiary |    100 | 2020-01-13
 Google     |    230 | 2020-01-14
 JD         |     10 | 2020-01-14
 Facebook   |    205 | 2020-01-13
 Caixin     |     13 | 2020-01-10
 Jiumodiary |    220 | 2020-01-15
 Facebook   |    201 | 2020-01-17
(8 rows)

```

#### **4 合集**  显示所有的，好不好都要。

```sql
SELECT A.name,B.acount,B.adate FROM site A FULL OUTER JOIN log B ON A.id=B.site_id;
    name    | acount |   adate    
------------+--------+------------
 Google     |     45 | 2020-01-10
 Jiumodiary |    100 | 2020-01-13
 Google     |    230 | 2020-01-14
 JD         |     10 | 2020-01-14
 Facebook   |    205 | 2020-01-13
 Caixin     |     13 | 2020-01-10
 Jiumodiary |    220 | 2020-01-15
            |    545 | 2020-01-16
 Facebook   |    201 | 2020-01-17
 wikipedia  |        | 
(10 rows)

```

#### **5 子集除外**  显示不完整的，有站名无记录+有记录无站名，求缺是也。

```sql
SELECT A.name,B.acount,B.adate from site A FULL OUTER JOIN log B ON A.id=B.site_id WHERE A.id IS NULL OR B.aid IS NULL;
   name    | acount |   adate    
-----------+--------+------------
           |    545 | 2020-01-16
 wikipedia |        | 
(2 rows)

```

#### **6 相对补集 A \ B**  显示有站名无记录的那个

```sql
SELECT A.name,B.acount,B.adate FROM site A LEFT JOIN log B ON A.id=B.site_id WHERE B.aid IS NULL;
   name    | acount | adate 
-----------+--------+-------
 wikipedia |        | 
(1 row)

```

#### **7 相对补集 B \ A** 显示有记录无站名的那个

```sql
SELECT A.name,B.acount,B.adate FROM site A RIGHT JOIN log B ON A.id=B.site_id WHERE A.id IS NULL;
 name | acount |   adate    
------+--------+------------
      |    545 | 2020-01-16
(1 row)
```

#### 还有其它可能吗？思考下哈


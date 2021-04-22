# UXDB实例

## JSON数据的操作

UXDB支持了JSON和JSONB相关函数，中间服务层（如PHP,JAVA）通过JDBC就可以直接获取到JSON数据，而无需再用org.json和json-lib库把以前的行数据进行转换。JSON还可以和普通表做关联查询，达到在同一数据库中结构化和半结构化的结合。我们用JSON来存储学生个人信息、课程信息、教师信息，用用传统关系表存储课程考试分数信息，通过对表与JSON的操作演示如何在UXDB中操作半结构化数据，如何连接Json字段和普通表。

### 创建表

```
uxdb=# create database abc;  /*建库*/
uxdb=# \c abc;  /*连接库*/
```

```sql
CREATE TABLE students
(
 studentid SERIAL primary key ,
 studentdesc jsonb
);
```

### 插入两条数据

```sql
insert into students(studentdesc) values('{  
   "sname":"李思佳",
   "sdata":{"sno":"1001","sage":"23","ssex":"女","tel":["010-82886998","13550011000"],"addr":["北京市海淀区中关村资本大厦","海淀区学院南路62号"]},
   "courses":[
      {"cno":"01","cname":"math","teacher":{"tno":"101","tname":"张三"}},
      {"cno":"02","cname":"chinese","teacher":{"tno":"102","tname":"李四"}},
      {"cno":"03","cname":"english","teacher":{"tno":"103","tname":"Steven"}}
   ],
"date":"2018-05-12"
}');

insert into students(studentdesc) values('{  
   "sname":"张静",
   "sdata":{"sno":"1004","sage":"20","ssex":"女","tel":["021-2861789","18211028796"],"addr":["科技一路6号","科技二路8号","科技三路20号"]},
   "courses":[
      {"cno":"01","cname":"math","teacher":{"tno":"101","tname":"张三"}},
      {"cno":"02","cname":"chinese","teacher":{"tno":"102","tname":"李四"}},
      {"cno":"03","cname":"english","teacher":{"tno":"103","tname":"Steven"}}
   ],
   "sc":[
      {"cno":"01","score":"76"},{"cno":"02","score":"77"},{"cno":"03","score":"87"}
   ],
   "date":"2018-05-21"
}');
```

### Json数据操作

#### 查询学生的基本信息：

```sql
select studentdesc->>'sdata' as sdata from students where studentid=1;
```

#### 查询学生年龄：

```sql
select studentdesc->'sdata'->>'sage'  as sage  from students where studentid=1;
```

#### 查询第一个地址信息：

```sql
select studentdesc->'sdata'#>>'{addr,0}'  as addr  from students where studentid=1;
```

#### 查询第一门课程信息：

```sql
select studentdesc#>'{courses,0}' as courses from students where studentid=1;
```

#### 查询第一门课程课程名：

```sql
select studentdesc#>'{courses,0}'->'cname'  as cname from students where studentid=1;
```

#### 查询第一门课程授课教师名：

```
select studentdesc#>'{courses,0}'->'teacher'->>'tname'  as cname from students where studentid=1;
```

#### 新增一个属性

```sql
update students set studentdesc=studentdesc|| '{"nickname":"大雄"}'  where studentid=1;
```

#### 删除一个属性

```sql
update students set studentdesc=studentdesc-'nickname'  where studentid=2;
```

#### 删除所有成绩

```sql
update students set studentdesc=studentdesc-'sc' ;
```

### 和普通表的联合查询

#### 创建表

```sql
create table sc(sno text,cno text,score text);
alter table sc add primary key(sno,cno);
```

#### 准备数据

```sql
insert into sc values('1001','01','99');
insert into sc values('1001','02','89');
insert into sc values('1001','03','77'); 
insert into sc values('1002','01','45');
insert into sc values('1002','02','78');
insert into sc values('1002','03','100'); 
insert into sc values('1003','01','90');
insert into sc values('1003','02','76');
insert into sc values('1003','03','87');
```

#### 表可以转为JSON

```sql
select row_to_json(sc.*) from sc where sno='1001'; 
```

可返回需要的各种SQL数据，从而简化中间层（JAVA/PHP）的代码

#### JSON可以和表做关联查询

```sql
select * from sc join students s on  text(s.studentdesc->'sdata'->>'sno')=sc.sno and sc.sno='1001' and sc.cno='01';
```


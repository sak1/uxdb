# UXDB实例

## UXDB逻辑复制实现多实例数据同步

UXDB逻辑复制可执行有选择性的复制，在不同的数据大版本之间进行复制，在不同平台间进行复制，备库写入操作，解决物理复制的局限性。逻辑复制采用的是=="发布-订阅"==模型，一个或多个订阅者可以订阅一个发布者上的一个或多个发布。订阅者最初从发布者那里收到复制的数据库对象的副本，并在同一对象上实时进行后续更改。这种数据复制方法又被称为事务复制。

### 建两个数据库实例

```
./initdb -W -D ../data1
./initdb -W -D ../data2
```

#### 修改实例A的uxsinodb.conf

```
port=5433
wal_level=Logical
```

#### 修改实例B的uxsinodb.conf

```
port=5434
wal_level=Logical
```

#### 启动

```
./ux_ctl restart -D ../data1
./ux_ctl restart -D ../data2
```

### 开两个窗口操作

#### A窗口，用于发布

```shell
./uxsql -p 5433
```

接下来，我们将展示如何使用逻辑复制。首先我创建一个简单的表，并插入一条记录。

```sql
create table t1 (a integer PRIMARY KEY, b integer);
insert into t1 values(1, 1);
```

接下来，我们为这个表创建一个发布。

```sql
create publication my_pub FOR table t1;
```

现在，发布已经创建完了，现在我们需要创建订阅。注意，我们在同一机器上的不通端口运行两个uxdb数据库实例。

#### B窗口，用于订阅

```shell
./uxsql -p 5434
```

```sql
create table t1(a integer PRIMARY KEY, b integer);
create SUBSCRIPTION my_sub CONNECTION 'host=localhost port=5433 dbname=uxdb password=mypassword' PUBLICATION my_pub;
```

现在，我们可以验证数据是否被复制过来。

```
TABLE t1;
```

接下来，我在验证数据更改之后是否被复制。首先，我们在发布者上插入一条记录，随后在订阅者上查看该记录是否被复制。

更新订阅

#### A窗口，更新数据

```sql
insert into t1 values(2, 2);
...
1 rows
```

#### B窗口，查看订阅数据是否被更新

```sql
table t1;
...
2 rows
```

从上面可以看到，数据被准确地复制到了订阅者。

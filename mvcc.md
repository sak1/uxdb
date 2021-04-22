# UXDB实例

## 事务隔离机制与并发解析

### 概述

数据一致性与性能，事务隔离与并发就象是跷跷板的两端，难以兼得；为了解决并发问题，第一招，引入“锁”，却拉低了性能，，第二招：多版本机制，但也带来了新的问题，如表膨胀。这就是"按倒了葫芦翘起了瓢"，遇山开路，遇水架桥吧。

## 简介

#### 1 非预期的读现象

首先，我们熟悉并发引发的非预期的读现象：脏读、不可重复读 、幻读、序列化异常

当第一个事务读取了第二个事务中已经修改但还未提交的数据，包括INSERT、UPDATE、 DELETE，当第二个事务不提交并执行ROLLBACK后，第一个事务所读取到的数据是不正确的，这种读现象称作脏读。是不是有点晕。人话就是：

1、脏读 ，读到了提交又回滚(rollback)了的数据，被放鸽子了；

2、不可重复读 ，读到了被修改或删除(update、delete)的数据，以致于两次读的结果不一样；

3、幻读 ，读到了新增加(insert)的数据，以致于两次读的结果不一样；

4、序列化异常，是指成功提交的一组事务的执行结果与这些事务按照串行执行方式的执行结果不一致。

#### 2 事务隔离级别

为了应对这些问题，数据库采用了4个级别的事务隔离，分别是、读未提交、读已提交、可重复读、可序列化

SQL标准定义的事务隔离级别与读现象的关系

| 隔离级别                      | 脏读   | 不可重复读 | 幻  读 | 序列化异常 |
| ----------------------------- | ------ | ---------- | ------ | ---------- |
| Read Uncommitted（读未提交）  | 可能   | 可能       | 可能   | 可能       |
| Read   Committed （读已提交） | 不可能 | 可能       | 可能   | 可能       |
| Repeatable Read （可重复读）  | 不可能 | 不可能     | 不可能 | 可能       |
| Serializable(可序列化)        | 不可能 | 不可能     | 不可能 | 可能       |

低级别的隔离级别支持更高的并发处理，系统开销更低，但增加了并发引发的副作用。 我们优先考虑Read Committed隔离级别。 它能够避免脏读，而且具有较好的并发性能。

我们看下系统默认的事务隔离级别是什么？

```sql
SELECT current_setting('transaction_isolation');
 current_setting 
-----------------
 read committed
(1 row)
```

果然是“read committed”——读已提交，就是说这种情况下不会脏读，其它是可能发生。

修改全局的事务隔离级别

```sql
alter system set default_transaction_isolation to 'repeatable read';
/*4个选项：serializable, repeatable read, read committed, read uncommitted*/
select ux_reload_conf();
\q
```

```shell
./ux_ctl restart -D ../data
#注意，改完您可能重启动失败，这时，请修改数据目录下的uxsinodb.auto.conf，你看到repeatable read中间少了个空格，原因待查。加个空格，再重新启动。
```

查看当前会话的事务隔离级别：

```shell
show transaction_isolation;
transaction_isolation 
-----------------------
 read uncommitted
(1 row)
```

设置当前会话的事务隔离级别

```
set session characteristics as transaction isolation level read uncommitted; 
```

### 第一招：锁——悲观机制

锁机制是一种预防性的机制，读会阻塞写，写也会阻塞读，基本的封锁类型有两种：排它锁和共享锁

- 排它锁：被加锁的对象只能被持有锁的事务读取和修改，其他事务无法在该对象上加其他锁，也不能读取和修改该对象。 
- 共享锁：被加锁的对象可以被持锁事务读取，但是不能被修改，其他事务也可以在上面再加共享锁。

### 第二招：多版本并发控制——乐观机制

基于锁的并发控制机制要么延迟一项操作，要么中止发出该操作的事务来保证可串行性。 如果每一数据项的旧值副本保存在系统中，这些问题就可以避免。 这种基于多个旧值版本的并发控制即MVCC。MVCC通过保存数据在某个时间点的快照，并控制元组（行记录）的可见性来实现。其它数据库如Oracle、 PostgreSQL、 MySQL(lnnodb）都使用MVCC并发控制机制。 

UXDB为每一个事务分配一个递增的、类型为int32的整型数作为唯一的事务ID，称为xid。xmin保存了创建该行数据的事务的xid, xmax保存的是删除该行的xid,

```
SELECT xmin,xmax, cmin, cmax, id,ival FROM  tbl_mvcc WHERE  id  =  1;
   xmin   |   xmax   | cmin | cmax | id | ival 
----------+----------+------+------+----+------
 49571583 | 49571584 |    0 |    0 |  1 |    1
(1 row)
```

通过pageinspect观察MVCC

```
CREATE EXTENSION pageinspect;
\dx+  pageinspect 
```

MVCC的旧版本和新版本存储在同一个地方，如果更新大量数据，将会导致数据表的膨胀。 

我们换个小节，讨论这个问题，见《如何解决表膨胀问题》
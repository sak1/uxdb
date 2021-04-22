# UXDB实例

## lo大对象

lo模块提供管理大对象（也被称为LO或BLOB）的支持。这包括一种数据类型lo以及一个触发器lo_manage。

#### 原理

JDBC驱动的问题之一是假定对BLOB的引用被存储在一个表中，如果该表被改变，相关的BLOB会被从数据库删除；ODBC驱动也存在这样的问题。

但对于UXDB这并不会发生。大对象被当做自主的对象，表对象通过OID引用大对象，可以有多个表对象通过OID引用同一个大对象，因此系统不会因为表对象的改变或者删除而删除大对象。但是由于使用JDBC或ODBC的标准代码不会删除大对象，从而导致孤立大对象—不被任何表引用的大对象，会始终占据磁盘空间。

lo模块允许通过附加一个触发器到包含LO引用列的表来修复这种问题。该触发器本质上只是在删除或修改一个引用大对象的值时做lo_unlink。使用这个触发器时，触发器控制的列中的大对象必须只有一个数据库引用。

这个模块也提供了一种数据类型lo，是oid类型的一个域。有助于区分包含大对象引用的数据库列和包含其他类的数据库列。并非必须使用lo类型来使用触发器，但是用它来追踪数据库中哪些列表示用触发器管理的大对象非常方便。

#### 用法示例

```
CREATE EXTENSION lo;
CREATE TABLE image (title TEXT, raster lo);
CREATE TRIGGER t_raster BEFORE UPDATE OR DELETE ON image
    FOR EACH ROW EXECUTE PROCEDURE lo_manage(raster);
```

对每一个将包含大对象唯一引用的列，创建一个BEFORE  UPDATE  OR  DELETE触发器，并把该列名作为唯一的触发器参数。也可以用BEFORE UPDATE OF column_name来限制该触发器只对该列上的更新事件执行。如果需要在同一个表中有多个lo列，为每一个lo列创建一个独立的触发器，记住为同一个表上的每个触发器指定不同的名称。

#### 限制

- 清空或删除表仍将让表中包含的大对象变成孤立的，因为触发器在这种情况下不会被执行。可以在TRUNCATE TABLE或DROP TABLE之前使用DELETE FROM table来避免这种问题。如果怀疑有孤立的大对象，可以使用客户端应用vacuumlo模块进行清理。
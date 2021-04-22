# UXDB实例

## 三种数据类型转换方法

格式化函数、CAST函数、操作符

### 1 格式化函数

- 把时间戳转换成字符串	to_char(current_timestamp,'HH12:MI:SS')

- 把间隔转换成字符串	to_char(interval'15h2m12s','HH24:MI:SS')

- 把整数转换成字符串	to_char(125,'999')

- 把数字转换成字符串	to_char(-125.8,'999D99S')

- 把字符串转换成日期	to_date('05Dec2000','DDMonYYYY')

- 把字符串转换成数字	to_number('12,8888','99G999D9')

- 把字符串转换成时间戳	to_timestamp('05 Dec 2000','DD Mon yyyy')

  您可以使用命令验证：如：

  ```
  select to_char(125,'999');
  ```

  

### 2 通过CAST函数进行转换

将varchar字符类型转换成text类型，如下所示：

```sql
SELECT CAST(varchar'123' as text); 
```

将varchar字符类型转换成int4类型，如下所示：

```sql
SELECT CAST(varchar'123' as int4 ); 
```


### 3 通过::操作符进行转换

转换成int4或numeric类型：

```
select 1::int4,3/2::numeric;
```

另一个例子，通过SQL查询给定表的字段名称，先根据表名在系统表ux_class找到表的OID，其中OID为隐藏的系统宇段：

```
select oid,relname from ux_class where relname='address';
```

之后根据test_json1 表的OID， 在系统表ux_attribute中根据attrelid （即表的OID）找到表的字段，如下所示：

```
SELECT attname  FROM ux_attribute WHERE  attrelid='49160' AND  attnum>0; 
```

上述操作需通过两步完成，但通过类型转换可一步到位， 如下所示：

```
select attname from ux_attribute where attrelid='address'::regclass and attnum >0;
```

第－种方法兼容性相对较好，第三种方法用法简捷。


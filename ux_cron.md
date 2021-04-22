# UXDB实例

## 定时任务管理器

ux_cron是一个简单的定时任务管理器，例如：可以进行一些定期向表中插入数据的操作。

### 安装配置

修改uxsinodb.conf配置文件，添加如下内容：

```shell
#shared_preload_libraries	=	''	#(change requires restart)
shared_preload_libraries	=	'ux_cron'
cron.database_name	=	'uxdb'
```

> 注意：配置了cron.database_name参数之后，才能在对应的数据库上加载ux_cron扩展，否则创建扩展会报错。

配置ux_hba.conf中密码验证方式为trust，也可以配置.uxpass文件。

```sh
#"local" is for Unix domain socket connections only
localhost	all		all		trust
```

启动数据库，创建ux_cron扩展。

```sql
./uxsql
create extension ux_cron;
```

### 定时设置。

例如：每隔1分钟向time_cron表中插入一条当前时间

1. 创建一张time_cron表。

   ```
   create table time_cron(data timestamp);
   ```

2. 建立定时任务，每分钟向time_cron表中插入一条当前时间记录。

   方式一：适用于本地数据库

   ```sql
   SELECT cron.schedule('*/1 * * * *', 'insert into time_cron values (now())'); 
   ```

   方式二：适用于远程数据库

   ```sql
   INSERT INTO cron.job (schedule, command, nodename, nodeport, database, username) 
   		VALUES ('*/1 * * * *', 'insert into time_cron values (now())', '127.0.0.1', 5432, 'uxdb', 'mypassword');
   ```

3. 查看任务是否创建成功。

   ```sql
   select * from cron.job;
   ```

4. 等待几分钟，查看time_cron表中数据。

   ```sql
   select * from time_cron;
   ```

5. 取消任务。

   方式一：

   ```sql
   SELECT cron.unschedule(2);
   ```

   方式二：

   ```sql
   delete from cron.job where jobid=2;
   ```
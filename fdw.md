创建外部表的示例。
1.首先安装该扩展：
CREATE EXTENSION uxdb_fdw;
2.使用CREATE  SERVER创建一个外部服务器（示例中连接的UXDB服务器主机IP：
192.83.123.89；端口：5432；数据库：foreign_db）：
CREATE SERVER foreign_server
        FOREIGN DATA WRAPPER uxdb_fdw
        OPTIONS (host '192.83.123.89', port '5432', dbname 'foreign_db');
3.用CREATE USER MAPPING定义一个用户映射来标识在远程服务器上使用哪个角色：
CREATE USER MAPPING FOR local_user
        SERVER foreign_server
        OPTIONS (user 'foreign_user', password 'password');
4.使用CREATE FOREIGN TABLE创建外部表（示例中访问远程服务器上的表名：
some_schema.some_table；它的本地名称：foreign_table）：
CREATE FOREIGN TABLE foreign_table (
        id integer NOT NULL,
        data text
)
        SERVER foreign_server
        OPTIONS (schema_name 'some_schema', table_name 'some_table');
CREATE  FOREIGN  TABLE中声明的列数据类型和其他特性必须要匹配实际的远程表。列名
也必须匹配，不过也可以为个别列附上column_name选项以表示它们在远程服务器上对应哪
个列。在很多情况中，要使用IMPORT FOREIGN SCHEMA手工构造外部表定义。
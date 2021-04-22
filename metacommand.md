# UXDB实例

## 元命令

元命令是uxsql 具有的功能，用户在不查询数据库的情况下实现复杂功能的命令。

注意：元命令登录库后，在SQL命令提示符下操作的，不是在linux提示符下操作的

### 通用

\copyright 	显示版权以及发布条款
\errverbose 	显示最近的服务器错误消息
\g /tmp/a.txt  	执行查询，并将结果发送到文件或管道
insert into city select * from city; \gexec  	执行查询，然后在结果中执行每个值
select count(*) from city;\gset  	执行查询并将结果存储在uxsql变量中
select * from city;\gx /tmp/a.txt 	类似\g，但是展开了输出模式
select * from city;\watch 1  	每SEC秒执行查询，默认2s
\? options 	显示反斜杠命令的帮助
\? 	显示有关uxsql命令行选项的帮助
\h select	显示SQL命令的语法帮助，*为所有命令
\q 	退出uxsql

### 查询缓冲区

\e	用外部编辑器编辑查询缓冲区（或文件）
\ef 	使用外部编辑器编辑function定义
\ev 	使用外部编辑器编辑view定义
\p 	显示查询缓冲区的内容
\r	重置（清除）查询缓冲区
\s 	显示历史操作记录，将其保存到文件

### 输入/输出

\copy abc to /tmp/tableabc.txt	用数据流执行SQL COPY到客户端主机
\\! cat /tmp/tableabc.txt	把字符打印到标准输出
\echo `date`  	输出日期  
\i 	从文件执行命令，如\i /tmp/sql.sql
\ir	命令类似于\i，\ir相对于脚本所在的目录而不是根据当前工作目录。
\o /tmp/sql.log	将所有查询结果发送到文件或管道
\qecho 类似echo，将输出结果发送到文件或管道

### 信息

> 选项：S =显示系统对象，+ =附加的详细信息

\dS列举表格视图和序列
\d+	列出表格视图和序列的描述信息
\daS 	列出聚集函数
\dA+ 	列出访问方法
\db[+] 	列出表空间
\dc[S+] 	列出字符集编码之间的转换。
\dC[+] 	列出类型转换
\dd[S] 	显示约束、操作符类、操作符族、规则以及触发器类型对象的描述。
\dD[S+] 	列出域。
\ddp 	列出默认的访问特权设置。
\dE[S+]	 列出外部表。
\det[+] 	列出外部表。
\des[+]	列出外部服务器。
\deu[+] 	列出用户映射。
\dew[+] 	列出外部数据包装器。
\df[antw][S+] 列出函数，以及它们的结果数据类型、参数数据类型和函数类型
\dF[+] 	列出文本搜索配置。
\dFd	列出文本搜索字典。
\dFp[+] 	列出文本搜索解析器。
\dFt[+] 	列出文本搜索模板。
\dg[S+] 	列出数据库角色，等同于与\du。
\di[S+] 	列出索引。
\dl 	显示大对象列表, 类似 \lo_list
\dL[S+] 	列出过程语言。
\dm[S+] 	列出物化视图。
\dn[S+]	 列出模式。
\do[S] 	列出操作符及其操作数和结果类型。
\dO[S+] 	列出排序规则。
\dp 	列出表、视图和序列，包括与它们相关的访问特权。
\drds 	列出已定义的配置设置。
\dRp[+] 	列出复制发布。
\dRs[+] 	列出复制订阅。
\ds[S+] 	列出序列。
\dt	列出表。
\dT	列出数据类型。
\du	列出数据库角色
\dv	列出视图。
\dx[+] 	列出已安装的扩展。
\dy	 列出事件触发器。
\l	列出数据库
\sf add	显示函数定义
\sv v_abc	显示视图定义
\z	列出表、视图和序列，包括与它们相关的访问特权。同\dp

### 格式化

\a 在未对齐和对齐的输出模式之间切换
\C nice	设置查询结果的任何表的标题，或者重置这类标题。
\f *	设置用于非对齐查询输出的域分隔符
\H 	开启HTML查询输出格式。
\pset 	设置影响查询结果表输出的选项。
\t 	只显示行（默认关闭）	\t [on|off] 
\T city 指定在HTML输出格式中，要放在table标签内的属性。
\x 设置或者切换扩展表格格式化模式。

### 连接

\c uxdb	连接到新数据库
\conninfo	输出有关当前数据库连接的信息。
\encoding	显示或设置客户端字符集编码
\password	更改指定用户的口令。

### 操作系统

\cd 修改当前工作目录
\cd /tmp
\\! pwd
\setenv CHA as 设置环境变量
\\!env |grep CHA 显示环境变量
\timing 显示每个 SQL 语句花费的时间（毫秒为单位），默认关闭。

### 变量

\prompt 提示用户提供一个文本用于分配给变量。
\set variable abc 设置变量variable为abc，\set列出所有变量
\unset variable 取消变量的设置

### 大对象

\lo_export LOBOID FILE	从数据库中读取具有OID的大对象，并把它写入到file。
\lo_import FILE [COMMENT]	把该文件存储到UXDB大对象。可选，它可以把给定的注释关联到该对象。
\lo_list	显示当前存储在数据库中的所有UXDB大对象，同时显示它们的任何注释。
\lo_unlink LOBOID large object operations 从数据库中删除OID为LOBOID的大对象。
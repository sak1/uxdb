# UXDB实例

## 身份验证配置详解

HBA（host-based authentication)表示==基于主机的身份验证==

用于客户端身份验证的控制配置文件，通常存在数据库的数据目录中。默认ux_hba.conf文件是在initdb初始化数据目录时安装的，如该文件位于初始化安装的数据库目录(如/home/uxdb/uxdbinstall/dbsql/data)下，也可以将身份验证配置文件放在其他位置

每个记录指定连接类型、客户端IP地址范围、数据库名称、用户名以及用于匹配这些参数的连接的身份验证方法。第一条记录用于执行身份验证。如果一条记录验证失败，则不考虑后续记录。如果没有匹配的记录，则拒绝访问。



编辑 ux_hba.conf 配置文件

```shell
vi ux_hba.conf
```

### TYPE 参数设置

TYPE 表示主机类型，值可能为：

local，使用本地unix套接字
host，使用TCP/IP连接（包括SSL和非SSL）
host，结合“IPv4地址”使用IPv4方式，结合“IPv6地址”则使用IPv6方式
hostssl，只能使用SSL TCP/IP连接
hostnossl，不能使用SSL TCP/IP连接。

### DATABASE 参数设置

DATABASE 表示数据库名称,值可能为：

1. all 所有数据库
2. sameuser/samerole 过时了
3. replication  物理复制连接（注意，并不特指某一数据库）

### USER 参数设置

USER 表示用户名称，值可以为：`all`,`一个用户名`，`一组用户名` ，多个用户时，用逗号隔开。


### ADDRESS 参数设置

该参数可以为 `主机名称` 或者`IP/32(IPV4) `或 `IP/128(IPV6)`


### METHOD 参数设置

参数值如下：

1. trust, 无条件地允许连接。  
2. reject, 拒绝认证  
3. md5, 常用的密码认证方式
4. password, 明文密码传送
5. scram-sha-256,执行SCRAM-SHA-256身份验证以验证用户的密码。
6. gss, 使用GSSAPI对用户进行身份验证。仅适用于TCP/IP连接。
7. sspi, 使用SSPI对用户进行身份验证。仅在Windows上可用。
8. ident,获取客户的操作系统名然后检查该用户是否允许以要求的数据库用户进行连接
9. peer, 从操作系统获取客户端的操作系统用户名，检查它是否与请求的数据库用户名匹配。仅适用于本地连接。
10. pam, 使用操作系统提供的可插入认证模块服务(PAM)来认证
11. ldap, 使用 LDAP 进行认证。
12. radius, 使用RADIUS服务进行身份验证。
13. cert, 使用SSL客户端证书进行身份验证。

#### 注意

> 修改该配置文件中的参数，必须重启数据库服务
>
> 若要允许其它IP地址访问该主机数据库，须修改uxsinodb.conf的listen_addresses` 为 `*`
>
> 重启使用ux_ctl reload命令或在SQL中执行 select ux_reload_conf()

### 例如：

```shell
# 允许本地系统上的任何用户通过 Unix 域套接字以任意数据库用户名连接到任意数据库（本地连接的默认值）。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     trust

# 相同的规则，但是使用本地环回 TCP/IP 连接。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             127.0.0.1/32            trust

# 和前一行相同，但是使用了一个独立的掩码列
# TYPE  DATABASE        USER            IP-ADDRESS      IP-MASK             METHOD
host    all             all             127.0.0.1       255.255.255.255     trust

#IPv6上相同的规则
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             ::1/128                 trust

# 使用主机名的相同规则（通常同时覆盖IPv4和IPv6）。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             localhost               trust

# 允许来自任意具有 IP 地址192.168.93.x 的主机上任意用户以 ident 为该连接所报告的相同用户名连接到数据库 "uxdb"（通常是操作系统用户名）。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    uxdb        all             192.168.93.0/24         ident

# 如果用户的口令被正确提供，允许来自主机192.168.12.10的任意用户连接到数据库"uxdb"。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    uxdb        all             192.168.12.10/32        scram-sha-256

# 如果用户的口令被正确提供，允许example.com 中主机上的任意用户连接到任意数据库。要求大多数用户使用SCRAM认证，但对用户'mike'例外，该用户使用不支持SCRAM认证的旧客户端。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             mike            .example.com            md5
host    all             all             .example.com            scram-sha-256

# 如果没有前面的"host"行，这两行将拒绝所有来自192.168.54.1的连接（因为那些项将首先被匹配），但是允许来自互联网其他任何地方的
# GSSAPI连接。零掩码导致主机IP地址中的所有位都不会被考虑，因此它匹配任意主机。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             192.168.54.1/32         reject
host    all             all             0.0.0.0/0               gss

# 允许来自192.168.x.x主机的用户连接到任意数据库，如果它们能够通过ident检查。例如，假设ident说用户是"bryanh"并且他要求以
# UXDB 用户"guest1"连接，如果在ux_ident.conf有一个映射"omicron"的选项说"bryanh"被允许以"guest1"连接，则该连接将被允许。
## TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             192.168.0.0/16          ident map=omicron

# 如果这些是本地连接的唯一三行，它们将允许本地用户只连接到它们自己的数据库（与其数据库用户名同名的数据库），不过管理员和角色"support"的成员除外（它们可以连接到所有数据库）。文件$UXDATA/admins 包含一个管理员名称的列表。在所有情况下都要求口令。
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   sameuser        all                                     md5
local   all             @admins                                 md5
local   all             +support                                md5

#上面的最后两行可以被整合为一行：
local   all             @admins,+support                        md5

# 数据库列也可以用列表和文件名：
local   db1,db2,@demodbs  all                                   md5
```
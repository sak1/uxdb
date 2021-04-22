# UXDB实例

## UXDB数据库加密详解

### 1	全数据库级别加密

全库加密采用AES方式对数据库文件在存储介质上的数据进行加密，用于对数据文件的保护。在数据离开内存之前进行加密（即数据在写入存储介质之前就是加密的），解密的时候也是将数据从存储读出到内存中进行解密的。加密数据是以一个block的大小（默认是32K）为单位进行的，加密仅涉及到数据而不是block的头部和metadata的组织。当开启全数据库加密时，有20%左右的性能损失。

### 2	列加密

UXDB支持对列进行加解密，加解密借助uxcrypto完成。用户需要指定密钥进行对敏感列的加密或者解密。加密时，需要将密钥和明文数据发送到UXDB端，UXDB进行加密并存储密文；解密时，数据在UXDB数据库端进行解密，然后将解密的结果发送给客户端。当进行加解密的时候，密钥和数据仅在UXDB端存在极短的时间并全部在内存中，用过及丢弃。

### 3	通信信道加密

UXDB可以设置服务端和客户端之间的数据传输是加密的。UXDB利用openssl的库实现这一要求，但前提条件是服务端和客户端都要安装openssl工具包。在服务端安装好openssl之后，就可以利用openssl指令生成一对私钥和证书，用以对数据进行加解密，然后再对配置文件稍作修改就可以了。同样的，UXDB数据库内部的数据引擎端与存储端之间的通信也可以设置基于openssl的加密，但前提条件也是数据库引擎端和分布式存储的DIR/MRC/OSD都要安装openssl工具包。

### 4	列加密的两种方法

#### 4.1	数据准备

创建外部模块

```
 create extension uxcrypto;  
```

创建测试表

```
create table test_user(id serial,username varchar(32),password text);
```

给测试表创建索引

```
create unique index idx_test_user_username on test_user using btree(username);
```

查看测试表结构

```
\d test_user
Table "public.test_user"
Column  |         Type          | Collation | Nullable |                Default                
----------+-----------------------+-----------+----------+---------------------------------------
 id       | integer               |           | not null | nextval('test_user_id_seq'::regclass)
 username | character varying(32) |           |          | 
 password | text                  |           |          | 
Indexes:
    "idx_test_user_username" UNIQUE, btree (username)
```

#### 4.2	md5 加密    

插入测试用户信息，使用md5加密方法给password列值进行数据加密。

```sql
insert into test_user(username,password) values  ('user1',md5('123456'));
insert into test_user(username,password) values  ('user2',md5('123456'));
```

查询用户信息

```
 select * From test_user;
 id | username |             password             
----+----------+----------------------------------
  1 | user1    | e10adc3949ba59abbe56e057f20f883e
  2 | user2    | e10adc3949ba59abbe56e057f20f883e
(2 rows)
```

认证测试，如果用户输入password的是123456，则通过认证。

```
select * From test_user where password=md5('123456');
 id | username |             password             
----+----------+----------------------------------
  1 | user1    | e10adc3949ba59abbe56e057f20f883e
  2 | user2    | e10adc3949ba59abbe56e057f20f883e
(2 rows)
```

备注：使用 md5 加密，密文相同，加密结果也相同，通过结果可推知密文，保密性不够强。

#### 4.3 crypt()函数加密

使用 crypt() 函数增加两条用户信息
插入测试用户信息，使用crypt加密方法给password列值进行数据加密。

```sql
insert into test_user(username,password) values('user3',crypt('123456',gen_salt('md5')));
insert into test_user(username,password) values('user4',crypt('123456',gen_salt('md5')));
/*gen_salt('md5')生成一个随机字符串，可接受des, xdes, md5 和 bf算法*/
```

查询新增用户信息

```sql
select * From test_user where username in ('user3','user4');
 id | username |              password              
----+----------+------------------------------------
  3 | user3    | $1$7wuaFG8c$2A7IIKlnO7M3DRGhmVGSB.
  4 | user4    | $1$8CePi7UZ$I/O0jCKSTa6ENxuXooLYX1
(2 rows)
```

认证测试，如果用户输入password的是123456，则通过认证。

```sql
select * From test_user where password=crypt('123456',password); 
 id | username |              password              
----+----------+------------------------------------
  3 | user3    | $1$7wuaFG8c$2A7IIKlnO7M3DRGhmVGSB.
  4 | user4    | $1$8CePi7UZ$I/O0jCKSTa6ENxuXooLYX1
(2 rows)
```

备注：虽然 user3，user4 使用相同的密码，但经过 crypt() 函数加密后，加密后的密码值不同，显然，crypt()函数加密比 md5 安全。

### uxcrypto加密组件

#### 5.1	计算hash值的函数

以下是几个计算Hash值的函数：

```
digest(data text, type text) returns bytea
digest(data bytea, type text) returns bytea
hmac(data text, key text, type text) returns bytea
hmac(data bytea, key text, type text) returns bytea
```

type为算法.支持 md5, sha1, sha224, sha256, sha384, sha512. 如果编译UXDB时有with-openssl选项, 还可以支持更多算法。

示例：

（1）相同函数调用多次，得到的结果一定相同：

```sql
select digest('Hello World!', 'md5');
​               digest   
 \xed076287532e86365e841e92bfc50d8c
(1 row)
```

（2）不同函数调用，得到的值是不相同的：

```sql
select digest('\xffffff'::bytea, 'md5');
​               digest               
 \x8597d4e7e65352a302b63e07bc01a7da
(1 row)
```

```sql
select digest('\xffffff', 'md5');
​               digest 
 \xd721f40e22920e0fd8ac7b13587aa92d
(1 row)
```

（3）hmac这两个函数与digest类似, 只是多了一个key参数, 也就是说同一个被加密的值, 可以使用不同的key得到不同的hash值.
这样的做法是, 不知道key的话, 也无法逆向破解原始值。使用hmac还有一个好处是, 使用digest如果原始值和hash值同时被别人修改了是无法知道是否被修改的，但是使用hmac, 如果原始值被修改了, 同时key没有泄漏的话, 那么hash值是无法被修改的, 因此就能够知道原始值是否被修改过。

```
select hmac('Hello World!', 'this is a key', 'md5');
                hmac                
------------------------------------
 \x40e139720a5aaa873c9dd044d1e4229f
(1 row)

select hmac('Hello World!', 'this is another key', 'md5');
                hmac                
------------------------------------
 \xc3ac2c35aec0bbc4f04de52c8b4057b4
(1 row)

```




#### 5.2	PGP加密函数

uxcrypto包中的pgp加解密函数，加密的PGP消息由2部分组成：

- 这个消息的会话密钥, 会话密钥可以是对称密钥或公钥

- 使用该会话密钥加密的数据, 可以选择加密选项

  例如：
  compress-algo, unicode-mode, cipher-algo, compress-level, convert-crlf, disable-mdc, enable-session-key
  使用对称密钥加解密的函数如下：

```
pgp_sym_encrypt(data text, psw text [, options text ]) returns bytea
pgp_sym_encrypt_bytea(data bytea, psw text [, options text ]) returns bytea
pgp_sym_decrypt(msg bytea, psw text [, options text ]) returns text
pgp_sym_decrypt_bytea(msg bytea, psw text [, options text ]) returns bytea
```

#### PGP 对称加密

使用对称密钥加密, 这里的对称密钥为'pwd'字符串 :

```
select pgp_sym_encrypt('Hello World！', 'pwd', 'cipher-algo=bf, compress-algo=2, compress-level=9');
​                                                                          pgp_sym_encrypt 
 \xc30d04040302e16069e78617ced27ad24201eb0fe2c798d9c92c893e37440769b33bad7eca4e275c62f4bd830e415955a2c4
7652442286e2b84bb78af23c648215177e4af77b5d46f3984752214eb486d0d387
(1 row)
```

#### 解密

```
select pgp_sym_decrypt('\xc30d04040302e16069e78617ced27ad24201eb0fe2c798d9c92c893e37440769b33bad7eca4e275c62f4bd830e415955a2c4
7652442286e2b84bb78af23c648215177e4af77b5d46f3984752214eb486d0d387'::bytea, 'pwd');
 pgp_sym_decrypt 
 Hello World！
(1 row)
```

#### 5.3 非对称加解密的函数

使用公钥私钥体系加解密的函数有：

```
pgp_pub_encrypt(data text, key bytea [, options text ]) returns bytea
pgp_pub_encrypt_bytea(data bytea, key bytea [, options text ]) returns bytea
pgp_pub_decrypt(msg bytea, key bytea [, psw text [, options text ]]) returns text
pgp_pub_decrypt_bytea(msg bytea, key bytea [, psw text [, options text ]]) returns bytea
```

#### 公钥加密举例

首先，需要生成一堆密钥，当然，这个可以有很多种办法，在这里就不累述了。
转换公钥为bytea :

```
select dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----Version: GnuPG v1.4.5 (GNU/Linux)
mQGiBFGgIDgRBADALXrWA4PyT+Mj6be2Jl0kMeXItZqdqOp5fKOYNpWT2LqKBa9YRFMDZHSS1MIfjyDEi07O+TKm98haHBowbHB00qMXW6z6VAxtqdUBt9b0L52OwkA9awsclpalPvLwDAQGpxlJh7aQJ0hHjQRgvfTqJpOoCF4WoMVVSin5Ox2P4wCgzcr7FtLhbysH/Axqmx6Oc3wG3FMD/ij4ES38IDNAoacCOuWlA6MeaxWzgVka1Zl/h2ktUVJOLeM0saF6Z570/RYYFASC5cVazCG9Gbq6a3WvCy6LW9hV5XZIOU1VBXYxaCP1sMeWgRSbVathdJMqbcz+kzqabiCVHDt5Q8k/TuIEvmkTnUbk1ca1GTxRJMhXYldZLDAcA/4hnPbjxUUtkQ9S6gP3Cih/8SOA/E4YIj5PK3S9nRc6OKZ9NiGVXmhqff8PA4TmX08DiVclDzsaxpmB1yYUc44/rhEZ53XxKfHzjvowNKEevBm+cPEc0U9a5VSFIBwfwuAAVbbfdmLKM+HcBsPc/3uRnNGRZX6mMvUAG9UCk50zUrQGZGlnb2FsiGAEExECACAFAlGgIDgCGwMGCwkIBwMCBBUCCAMEFgIDAQIeAQIXgAAKCRDaMgsjY0+RL6mpAJ97/he5t1uatNKyO5v00bfuq9hMrQCgyihwLNzCn3immQp1E8fsg0Rfmaq5Ag0EUaAgORAIAPf4AB8dn2tYQlqOu3cC4g+yD95RV+lxERWLwMNzH1aVuSmBQzZA7pZuAyt9Joy6kwgkcilWtP1XgwUQBnyfS5QOiqNAbDoKFZWxLVKPpvb8jk20zbiWvqH9IUVlMaRjJrY1kdC1ckPFmx17k4uFU9GKFrfx/VukaTBhK3iByUD4JGUvmqKJvu3DX3JliIH4PJaBFxp/nOIy/66gPPz2DkSTBBNYTNVtMqkDLz2gcAWQ99TWsEXRskehPk9FSUDvrs62vC7ZsGmwihSMt/B/gQHc9rmEs5RkqNfyYKUoI4d5UCQdWnjI+V5Sppq6HQhRJ9M0ytoqmYJDKioKR0ewBQMAAwUH/jOZn7ei1rZLv0RP0Y+/E0ROkzpNmMuP+mNvZNrf/PCd1SvPxFZ2MNnhB0JN9a1OjJD8otqqvxMujyTx5z0RqD+7mWKb/q96NpG+fApZNGt6YiTc4a9FV9jpf+fYZyfpOj/bmPpHIUtheGzx/+WIL9gHWDiFR0nP9uXZoDZotuPqEsH2acoIE4oB4lLBvajuDtwnAZlajHMgXZD9W/xzdAlR5frfGNVIdvylwN2SOfSavl4VM4hG1uFc2J4szmivK0TesP1UcIdxnTlTvFieEqaP2rpG6WfVVO7N5ZiWXYOuazzRSEtfTjnGZRnx+WmkUb5KNvVhSg8F7oB7WrKMH/2ISQQYEQIACQUCUaAgOQIbDAAKCRDaMgsjY0+RLx/TAJ4uAleVExWDEVSbNeqm9wBkgRNGbgCeK648ARQH8pBNHcX/hsefvah7TO4==XGEV-----END PGP PUBLIC KEY BLOCK-----');
```

得到

```
\x9901a20451a02038110400c02d7ad60383f24fe323e9b7b6265d2431e5c8b59a9da8ea797ca398369593d8ba8a05af58445303647492d4c21f8f20c48b4ecef932a6f7c85a1c1a306c7074d2a3175bacfa540c6da9d501b7d6f42f9d8ec2403d6b0b1c9696a53ef2f00c0406a7194987b6902748478d0460bdf4ea2693a8085e16a0c5554a29f93b1d8fe300a0cdcafb16d2e16f2b07fc0c6a9b1e8e737c06dc5303fe28f8112dfc203340a1a7023ae5a503a31e6b15b381591ad5997f87692d51524e2de334b1a17a679ef4fd1618140482e5c55acc21bd19baba6b75af0b2e8b5bd855e57648394d550576316823f5b0c79681149b55ab6174932a6dccfe933a9a6e20951c3b7943c93f4ee204be69139d46e4d5c6b5193c5124c8576257592c301c03fe219cf6e3c5452d910f52ea03f70a287ff12380fc4e18223e4f2b74bd9d173a38a67d3621955e686a7dff0f0384e65f4f038957250f3b1ac69981d72614738e3fae1119e775f129f1f38efa3034a11ebc19be70f11cd14f5ae55485201c1fc2e00055b6df7662ca33e1dc06c3dcff7b919cd191657ea632f5001bd502939d3352b4066469676f616c8860041311020020050251a02038021b03060b090807030204150208030416020301021e01021780000a0910da320b23634f912fa9a9009f7bfe17b9b75b9ab4d2b23b9bf4d1b7eeabd84cad00a0ca28702cdcc29f78a6990a7513c7ec83445f99aab9020d0451a02039100800f7f8001f1d9f6b58425a8ebb7702e20fb20fde5157e97111158bc0c3731f5695b92981433640ee966e032b7d268cba930824722956b4fd57830510067c9f4b940e8aa3406c3a0a1595b12d528fa6f6fc8e4db4cdb896bea1fd21456531a46326b63591d0b57243c59b1d7b938b8553d18a16b7f1fd5ba46930612b7881c940f824652f9aa289beedc35f72658881f83c9681171a7f9ce232ffaea03cfcf60e44930413584cd56d32a9032f3da0700590f7d4d6b045d1b247a13e4f454940efaeceb6bc2ed9b069b08a148cb7f07f8101dcf6b984b39464a8d7f260a52823877950241d5a78c8f95e52a69aba1d085127d334cada2a9982432a2a0a4747b0050300030507fe33999fb7a2d6b64bbf444fd18fbf13444e933a4d98cb8ffa636f64dadffcf09dd52bcfc4567630d9e107424df5ad4e8c90fca2daaabf132e8f24f1e73d11a83fbb99629bfeaf7a3691be7c0a59346b7a6224dce1af4557d8e97fe7d86727e93a3fdb98fa47214b61786cf1ffe5882fd8075838854749cff6e5d9a03668b6e3ea12c1f669ca08138a01e252c1bda8ee0edc2701995a8c73205d90fd5bfc73740951e5fadf18d54876fca5c0dd9239f49abe5e15338846d6e15cd89e2cce68af2b44deb0fd547087719d3953bc589e12a68fdaba46e967d554eecde598965d83ae6b3cd1484b5f4e39c66519f1f969a451be4a36f5614a0f05ee807b5ab28c1ffd8849041811020009050251a02039021b0c000a0910da320b23634f912f1fd3009e2e02579513158311549b35eaa6f700648113466e009e2bae3c011407f2904d1dc5ff86c79fbda87b4cee
```

转换私钥为bytea：

```
select dearmor('-----BEGIN PGP PRIVATE KEY BLOCK-----
Version: GnuPG v1.4.5 (GNU/Linux)

lQG7BFGgIDgRBADALXrWA4PyT+Mj6be2Jl0kMeXItZqdqOp5fKOYNpWT2LqKBa9Y
RFMDZHSS1MIfjyDEi07O+TKm98haHBowbHB00qMXW6z6VAxtqdUBt9b0L52OwkA9
awsclpalPvLwDAQGpxlJh7aQJ0hHjQRgvfTqJpOoCF4WoMVVSin5Ox2P4wCgzcr7
FtLhbysH/Axqmx6Oc3wG3FMD/ij4ES38IDNAoacCOuWlA6MeaxWzgVka1Zl/h2kt
UVJOLeM0saF6Z570/RYYFASC5cVazCG9Gbq6a3WvCy6LW9hV5XZIOU1VBXYxaCP1
sMeWgRSbVathdJMqbcz+kzqabiCVHDt5Q8k/TuIEvmkTnUbk1ca1GTxRJMhXYldZ
LDAcA/4hnPbjxUUtkQ9S6gP3Cih/8SOA/E4YIj5PK3S9nRc6OKZ9NiGVXmhqff8P
A4TmX08DiVclDzsaxpmB1yYUc44/rhEZ53XxKfHzjvowNKEevBm+cPEc0U9a5VSF
IBwfwuAAVbbfdmLKM+HcBsPc/3uRnNGRZX6mMvUAG9UCk50zUgAAn2j3pAZqpbN3
xfFFL2YYhP+ePmrnCzm0BmRpZ29hbIhgBBMRAgAgBQJRoCA4AhsDBgsJCAcDAgQV
AggDBBYCAwECHgECF4AACgkQ2jILI2NPkS+pqQCfe/4XubdbmrTSsjub9NG37qvY
TK0AoMoocCzcwp94ppkKdRPH7INEX5mqnQI9BFGgIDkQCAD3+AAfHZ9rWEJajrt3
AuIPsg/eUVfpcREVi8DDcx9WlbkpgUM2QO6WbgMrfSaMupMIJHIpVrT9V4MFEAZ8
n0uUDoqjQGw6ChWVsS1Sj6b2/I5NtM24lr6h/SFFZTGkYya2NZHQtXJDxZsde5OL
hVPRiha38f1bpGkwYSt4gclA+CRlL5qiib7tw19yZYiB+DyWgRcaf5ziMv+uoDz8
9g5EkwQTWEzVbTKpAy89oHAFkPfU1rBF0bJHoT5PRUlA767Otrwu2bBpsIoUjLfw
f4EB3Pa5hLOUZKjX8mClKCOHeVAkHVp4yPleUqaauh0IUSfTNMraKpmCQyoqCkdH
sAUDAAMFB/4zmZ+3ota2S79ET9GPvxNETpM6TZjLj/pjb2Ta3/zwndUrz8RWdjDZ
4QdCTfWtToyQ/KLaqr8TLo8k8ec9Eag/u5lim/6vejaRvnwKWTRremIk3OGvRVfY
6X/n2Gcn6To/25j6RyFLYXhs8f/liC/YB1g4hUdJz/bl2aA2aLbj6hLB9mnKCBOK
AeJSwb2o7g7cJwGZWoxzIF2Q/Vv8c3QJUeX63xjVSHb8pcDdkjn0mr5eFTOIRtbh
XNieLM5orytE3rD9VHCHcZ05U7xYnhKmj9q6Ruln1VTuzeWYll2Drms80UhLX045
xmUZ8flppFG+Sjb1YUoPBe6Ae1qyjB/9AAFUCNmPaOMHsDg6bZpo7ApZJbJsiW6Z
BTCojWYF4E2C+5bPQoietziIE5V6LhPdiEkEGBECAAkFAlGgIDkCGwwACgkQ2jIL
I2NPkS8f0wCgpSCnhRw+soY+Fpg5t7IHjusqNt0An0EeMye7x5uyQc2ikmVtgfZu
NOxi
=FwrW
-----END PGP PRIVATE KEY BLOCK-----');
```

得到

```
\x9501bb0451a02038110400c02d7ad60383f24fe323e9b7b6265d2431e5c8b59a9da8ea797ca398369593d8ba8a05af58445303647492d4c21f8f20c48b4ecef932a6f7c85a1c1a306c7074d2a3175bacfa540c6da9d501b7d6f42f9d8ec2403d6b0b1c9696a53ef2f00c0406a7194987b6902748478d0460bdf4ea2693a8085e16a0c5554a29f93b1d8fe300a0cdcafb16d2e16f2b07fc0c6a9b1e8e737c06dc5303fe28f8112dfc203340a1a7023ae5a503a31e6b15b381591ad5997f87692d51524e2de334b1a17a679ef4fd1618140482e5c55acc21bd19baba6b75af0b2e8b5bd855e57648394d550576316823f5b0c79681149b55ab6174932a6dccfe933a9a6e20951c3b7943c93f4ee204be69139d46e4d5c6b5193c5124c8576257592c301c03fe219cf6e3c5452d910f52ea03f70a287ff12380fc4e18223e4f2b74bd9d173a38a67d3621955e686a7dff0f0384e65f4f038957250f3b1ac69981d72614738e3fae1119e775f129f1f38efa3034a11ebc19be70f11cd14f5ae55485201c1fc2e00055b6df7662ca33e1dc06c3dcff7b919cd191657ea632f5001bd502939d335200009f68f7a4066aa5b377c5f1452f661884ff9e3e6ae70b39b4066469676f616c8860041311020020050251a02038021b03060b090807030204150208030416020301021e01021780000a0910da320b23634f912fa9a9009f7bfe17b9b75b9ab4d2b23b9bf4d1b7eeabd84cad00a0ca28702cdcc29f78a6990a7513c7ec83445f99aa9d023d0451a02039100800f7f8001f1d9f6b58425a8ebb7702e20fb20fde5157e97111158bc0c3731f5695b92981433640ee966e032b7d268cba930824722956b4fd57830510067c9f4b940e8aa3406c3a0a1595b12d528fa6f6fc8e4db4cdb896bea1fd21456531a46326b63591d0b57243c59b1d7b938b8553d18a16b7f1fd5ba46930612b7881c940f824652f9aa289beedc35f72658881f83c9681171a7f9ce232ffaea03cfcf60e44930413584cd56d32a9032f3da0700590f7d4d6b045d1b247a13e4f454940efaeceb6bc2ed9b069b08a148cb7f07f8101dcf6b984b39464a8d7f260a52823877950241d5a78c8f95e52a69aba1d085127d334cada2a9982432a2a0a4747b0050300030507fe33999fb7a2d6b64bbf444fd18fbf13444e933a4d98cb8ffa636f64dadffcf09dd52bcfc4567630d9e107424df5ad4e8c90fca2daaabf132e8f24f1e73d11a83fbb99629bfeaf7a3691be7c0a59346b7a6224dce1af4557d8e97fe7d86727e93a3fdb98fa47214b61786cf1ffe5882fd8075838854749cff6e5d9a03668b6e3ea12c1f669ca08138a01e252c1bda8ee0edc2701995a8c73205d90fd5bfc73740951e5fadf18d54876fca5c0dd9239f49abe5e15338846d6e15cd89e2cce68af2b44deb0fd547087719d3953bc589e12a68fdaba46e967d554eecde598965d83ae6b3cd1484b5f4e39c66519f1f969a451be4a36f5614a0f05ee807b5ab28c1ffd00015408d98f68e307b0383a6d9a68ec0a5925b26c896e990530a88d6605e04d82fb96cf42889eb7388813957a2e13dd8849041811020009050251a02039021b0c000a0910da320b23634f912f1fd300a0a520a7851c3eb2863e169839b7b2078eeb2a36dd009f411e3327bbc79bb241cda292656d81f66e34ec62
```

使用公钥加密字符串'Hello World!'

```
select pgp_pub_encrypt('Hello World!', \x9901a20451a02038110400c02d7ad60383f24fe323e9b7b6265d2431e5c8b59a9da8ea797ca398369593d8ba8a05af58445303647492d4c21f8f20c48b4ecef932a6f7c85a1c1a306c7074d2a3175bacfa540c6da9d501b7d6f42f9d8ec2403d6b0b1c9696a53ef2f00c0406a7194987b6902748478d0460bdf4ea2693a8085e16a0c5554a29f93b1d8fe300a0cdcafb16d2e16f2b07fc0c6a9b1e8e737c06dc5303fe28f8112dfc203340a1a7023ae5a503a31e6b15b381591ad5997f87692d51524e2de334b1a17a679ef4fd1618140482e5c55acc21bd19baba6b75af0b2e8b5bd855e57648394d550576316823f5b0c79681149b55ab6174932a6dccfe933a9a6e20951c3b7943c93f4ee204be69139d46e4d5c6b5193c5124c8576257592c301c03fe219cf6e3c5452d910f52ea03f70a287ff12380fc4e18223e4f2b74bd9d173a38a67d3621955e686a7dff0f0384e65f4f038957250f3b1ac69981d72614738e3fae1119e775f129f1f38efa3034a11ebc19be70f11cd14f5ae55485201c1fc2e00055b6df7662ca33e1dc06c3dcff7b919cd191657ea632f5001bd502939d3352b4066469676f616c8860041311020020050251a02038021b03060b090807030204150208030416020301021e01021780000a0910da320b23634f912fa9a9009f7bfe17b9b75b9ab4d2b23b9bf4d1b7eeabd84cad00a0ca28702cdcc29f78a6990a7513c7ec83445f99aab9020d0451a02039100800f7f8001f1d9f6b58425a8ebb7702e20fb20fde5157e97111158bc0c3731f5695b92981433640ee966e032b7d268cba930824722956b4fd57830510067c9f4b940e8aa3406c3a0a1595b12d528fa6f6fc8e4db4cdb896bea1fd21456531a46326b63591d0b57243c59b1d7b938b8553d18a16b7f1fd5ba46930612b7881c940f824652f9aa289beedc35f72658881f83c9681171a7f9ce232ffaea03cfcf60e44930413584cd56d32a9032f3da0700590f7d4d6b045d1b247a13e4f454940efaeceb6bc2ed9b069b08a148cb7f07f8101dcf6b984b39464a8d7f260a52823877950241d5a78c8f95e52a69aba1d085127d334cada2a9982432a2a0a4747b0050300030507fe33999fb7a2d6b64bbf444fd18fbf13444e933a4d98cb8ffa636f64dadffcf09dd52bcfc4567630d9e107424df5ad4e8c90fca2daaabf132e8f24f1e73d11a83fbb99629bfeaf7a3691be7c0a59346b7a6224dce1af4557d8e97fe7d86727e93a3fdb98fa47214b61786cf1ffe5882fd8075838854749cff6e5d9a03668b6e3ea12c1f669ca08138a01e252c1bda8ee0edc2701995a8c73205d90fd5bfc73740951e5fadf18d54876fca5c0dd9239f49abe5e15338846d6e15cd89e2cce68af2b44deb0fd547087719d3953bc589e12a68fdaba46e967d554eecde598965d83ae6b3cd1484b5f4e39c66519f1f969a451be4a36f5614a0f05ee807b5ab28c1ffd8849041811020009050251a02039021b0c000a0910da320b23634f912f1fd3009e2e02579513158311549b35eaa6f700648113466e009e2bae3c011407f2904d1dc5ff86c79fbda87b4cee');
```

得到

```
\xc1c14e031b437bccd670d8451007fe2d921731879c4044946d8f5fc88fb40d4701895534d8d370882ac22b8ea899b014df1a835d3d5520a681c184816acef966e9a57201810eee18136dd8a148d81811de3f39ebaa7ada4e022fa3ce0f5d62937cad62ef28dbef8864f78c48f56ae932e8b2361d74dd3792388689e1443a77a06b81aa4a0bf0cb1d3158130c097d4f22483609e8c317914f3b99966d9274856884894917a1d0b1389ad62d303600edaed7edba52b9fd3890386fa9579089dbbe60124ba5dd211a710fb352e7400e2b13c139b6b9a7fc03308852ac19447afd01e660be6a72b4a2c0985f8abc66fbb874455c6f83cec73b51269538be7cc43229c37ae5658102c02c44e73cd60866550800dbbb466956161875455f5554db33b52b6897d78c8613a75d6627cd7cc5be2f5685bc67bcce04f2d9aef40404bd10ac1a5b3510c60c9b5220f2e8c87564ca133529282c04766aa902dddd19b2da3ddd8b5d95991276c4bcff6c720fa32522898567bb93bacef655f35c6b92bf72d6f75442b37f3c5fa05b1c903f25ff470e69340c946a2d985d2f384c7c1f99f7df8f0189366189af895d1615d6974adafeea2519e27b5b82a5d094a20fb8215bf7ed3f0f58c58b5fbe331da0bcfd49bf94426ba2e5db02b43a54ac9046fb11238e010b10cf4d9fec036ce1abb695b5462934f968682da6861583b58695b312a3d511b5e8091f6668ad116199567b453d7f51dfd23c0142070d0559e7e63c8ca3ed9363937184b90e273948e3dc80f121074f0f0b01f980a2d534683fc4b45131380cd70a564f33b72e6c812d858695dbb4


```

使用私钥解密这串bytea.

```
select pgp_pub_decrypt('\xc1c14e031b437bccd670d8451007fe2d921731879c4044946d8f5fc88fb40d4701895534d8d370882ac22b8ea899b014df1a835d3d5520a681c184816acef966e9a57201810eee18136dd8a148d81811de3f39ebaa7ada4e022fa3ce0f5d62937cad62ef28dbef8864f78c48f56ae932e8b2361d74dd3792388689e1443a77a06b81aa4a0bf0cb1d3158130c097d4f22483609e8c317914f3b99966d9274856884894917a1d0b1389ad62d303600edaed7edba52b9fd3890386fa9579089dbbe60124ba5dd211a710fb352e7400e2b13c139b6b9a7fc03308852ac19447afd01e660be6a72b4a2c0985f8abc66fbb874455c6f83cec73b51269538be7cc43229c37ae5658102c02c44e73cd60866550800dbbb466956161875455f5554db33b52b6897d78c8613a75d6627cd7cc5be2f5685bc67bcce04f2d9aef40404bd10ac1a5b3510c60c9b5220f2e8c87564ca133529282c04766aa902dddd19b2da3ddd8b5d95991276c4bcff6c720fa32522898567bb93bacef655f35c6b92bf72d6f75442b37f3c5fa05b1c903f25ff470e69340c946a2d985d2f384c7c1f99f7df8f0189366189af895d1615d6974adafeea2519e27b5b82a5d094a20fb8215bf7ed3f0f58c58b5fbe331da0bcfd49bf94426ba2e5db02b43a54ac9046fb11238e010b10cf4d9fec036ce1abb695b5462934f968682da6861583b58695b312a3d511b5e8091f6668ad116199567b453d7f51dfd23c0142070d0559e7e63c8ca3ed9363937184b90e273948e3dc80f121074f0f0b01f980a2d534683fc4b45131380cd70a564f33b72e6c812d858695dbb4','\x9501bb0451a02038110400c02d7ad60383f24fe323e9b7b6265d2431e5c8b59a9da8ea797ca398369593d8ba8a05af58445303647492d4c21f8f20c48b4ecef932a6f7c85a1c1a306c7074d2a3175bacfa540c6da9d501b7d6f42f9d8ec2403d6b0b1c9696a53ef2f00c0406a7194987b6902748478d0460bdf4ea2693a8085e16a0c5554a29f93b1d8fe300a0cdcafb16d2e16f2b07fc0c6a9b1e8e737c06dc5303fe28f8112dfc203340a1a7023ae5a503a31e6b15b381591ad5997f87692d51524e2de334b1a17a679ef4fd1618140482e5c55acc21bd19baba6b75af0b2e8b5bd855e57648394d550576316823f5b0c79681149b55ab6174932a6dccfe933a9a6e20951c3b7943c93f4ee204be69139d46e4d5c6b5193c5124c8576257592c301c03fe219cf6e3c5452d910f52ea03f70a287ff12380fc4e18223e4f2b74bd9d173a38a67d3621955e686a7dff0f0384e65f4f038957250f3b1ac69981d72614738e3fae1119e775f129f1f38efa3034a11ebc19be70f11cd14f5ae55485201c1fc2e00055b6df7662ca33e1dc06c3dcff7b919cd191657ea632f5001bd502939d335200009f68f7a4066aa5b377c5f1452f661884ff9e3e6ae70b39b4066469676f616c8860041311020020050251a02038021b03060b090807030204150208030416020301021e01021780000a0910da320b23634f912fa9a9009f7bfe17b9b75b9ab4d2b23b9bf4d1b7eeabd84cad00a0ca28702cdcc29f78a6990a7513c7ec83445f99aa9d023d0451a02039100800f7f8001f1d9f6b58425a8ebb7702e20fb20fde5157e97111158bc0c3731f5695b92981433640ee966e032b7d268cba930824722956b4fd57830510067c9f4b940e8aa3406c3a0a1595b12d528fa6f6fc8e4db4cdb896bea1fd21456531a46326b63591d0b57243c59b1d7b938b8553d18a16b7f1fd5ba46930612b7881c940f824652f9aa289beedc35f72658881f83c9681171a7f9ce232ffaea03cfcf60e44930413584cd56d32a9032f3da0700590f7d4d6b045d1b247a13e4f454940efaeceb6bc2ed9b069b08a148cb7f07f8101dcf6b984b39464a8d7f260a52823877950241d5a78c8f95e52a69aba1d085127d334cada2a9982432a2a0a4747b0050300030507fe33999fb7a2d6b64bbf444fd18fbf13444e933a4d98cb8ffa636f64dadffcf09dd52bcfc4567630d9e107424df5ad4e8c90fca2daaabf132e8f24f1e73d11a83fbb99629bfeaf7a3691be7c0a59346b7a6224dce1af4557d8e97fe7d86727e93a3fdb98fa47214b61786cf1ffe5882fd8075838854749cff6e5d9a03668b6e3ea12c1f669ca08138a01e252c1bda8ee0edc2701995a8c73205d90fd5bfc73740951e5fadf18d54876fca5c0dd9239f49abe5e15338846d6e15cd89e2cce68af2b44deb0fd547087719d3953bc589e12a68fdaba46e967d554eecde598965d83ae6b3cd1484b5f4e39c66519f1f969a451be4a36f5614a0f05ee807b5ab28c1ffd00015408d98f68e307b0383a6d9a68ec0a5925b26c896e990530a88d6605e04d82fb96cf42889eb7388813957a2e13dd8849041811020009050251a02039021b0c000a0910da320b23634f912f1fd300a0a520a7851c3eb2863e169839b7b2078eeb2a36dd009f411e3327bbc79bb241cda292656d81f66e34ec62');
```

```
 pgp_pub_decrypt 
 Hello World!
(1 row)
```

使用pgp_key_id可以在加密后的数据中取出这份数据是通过对称密钥还是公钥加密的,例如:

```
select pgp_key_id('\xc30d0404030245811e051118cc136ed23f0198808f069b53264d4a08c2b5dcf3b1c39a34d091263f7f6b64a14808e6ffb32ccc09749105b9cc062d70c628357ab1e2474ff6d109dd083ce892cfa55706');
```

```
 pgp_key_id 

 SYMKEY
(1 row)
```

```
select pgp_key_id('\xc1c14e031b437bccd670d8451007fe2d921731879c4044946d8f5fc88fb40d4701895534d8d370882ac22b8ea899b014df1a835d3d5520a681c184816acef966e9a57201810eee18136dd8a148d81811de3f39ebaa7ada4e022fa3ce0f5d62937cad62ef28dbef8864f78c48f56ae932e8b2361d74dd3792388689e1443a77a06b81aa4a0bf0cb1d3158130c097d4f22483609e8c317914f3b99966d9274856884894917a1d0b1389ad62d303600edaed7edba52b9fd3890386fa9579089dbbe60124ba5dd211a710fb352e7400e2b13c139b6b9a7fc03308852ac19447afd01e660be6a72b4a2c0985f8abc66fbb874455c6f83cec73b51269538be7cc43229c37ae5658102c02c44e73cd60866550800dbbb466956161875455f5554db33b52b6897d78c8613a75d6627cd7cc5be2f5685bc67bcce04f2d9aef40404bd10ac1a5b3510c60c9b5220f2e8c87564ca133529282c04766aa902dddd19b2da3ddd8b5d95991276c4bcff6c720fa32522898567bb93bacef655f35c6b92bf72d6f75442b37f3c5fa05b1c903f25ff470e69340c946a2d985d2f384c7c1f99f7df8f0189366189af895d1615d6974adafeea2519e27b5b82a5d094a20fb8215bf7ed3f0f58c58b5fbe331da0bcfd49bf94426ba2e5db02b43a54ac9046fb11238e010b10cf4d9fec036ce1abb695b5462934f968682da6861583b58695b312a3d511b5e8091f6668ad116199567b453d7f51dfd23c0142070d0559e7e63c8ca3ed9363937184b90e273948e3dc80f121074f0f0b01f980a2d534683fc4b45131380cd70a564f33b72e6c812d858695dbb4');
```

```
    pgp_key_id    

 1B437BCCD670D845
(1 row)
```



#### 5.4	其他函数

（1）bytea和ASCII-armor格式转化

```
armor(data bytea) returns text
dearmor(data text) returns bytea
```

（2）产生随机数加密数据bytea的函数

```
gen_random_bytes(count integer) returns bytea
```

#### 5.5	小结

- crypt+gen_salt 加密速度慢, 被破解的难度高. 密码字符串通常较短, 并且有非常高的安全要求, 所以比较适合用于对密码字段进行加密.
- 对称加密则比较适合数据量较大, 并且有安全需求的加密. 通常为了使用方便加密的密钥可以存储在本地库中, 为了提高安全, 也可以把加密的密钥存储在客户端或者其他数据库中.
- 公钥加密手段与对称加密类似, 只是需要生成钥匙对. 使用公钥加密后的数据解密需要用到对应的私钥. 如果私钥加了passphrase保护, 解密时还需要提供私钥的passphrase.


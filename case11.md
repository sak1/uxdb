# UXDB实例

## 我国移动基站地震风险规避示例

### 概述
移动基站选址与安全运维，关乎通信安全保障，降低建设与维护成本。
本例以2016年中国基站与中国地震数据为例，展示2016全国基站面临的地震风险。为基站建设提供参考。

数据来源于半官方数据，演示为主，仅供参考，正式使用需要对接专业数据。
### 实验
#### 1、将下载数据[地震](data/dizhen.rar)、[基站](jizhan.rar)两份数据导入到UXDB数据库中（模拟真实数据）

(1).在数据库中新建表，字段与Excel顺序相同（或者有多余字段在最后）；

(2).处理Excel表格```，先将Excel表格另存为csv文件，再将csv文件用记事本打开，另存为编码为UTF-8格式的csv文件。csv文件要带字段名。

(3).导入数据。在uxdbAdmin客户端中，点击工具栏“执行任意的SQL查询”按钮（有SQL字样的那个按钮），输入：
```copy jizhan(dian,ln,la)
from 'd:/jizhan.csv'
with (format csv,header true,quote '"',DELIMITER ',',encoding 'UTF_8');
```
(4)重复类似步骤，导入地震数据到dizhen表。

#### 2、使用baidu BDP，连接数据库。
#### 3、建两个图层，分别选择经纬度与地震等级等。展示结果。

![图一](D:\github\sak1\uxsinodb\img\2016.png)

![图二](D:\github\sak1\uxsinodb\img\2016p.png)

备注：基站数据可阶段导入，地震数据建议长期实时导入。
### 小结
地震数据仅代表可能产生灾害的可能性，不代表一定发生。如2016年造成破坏性的4.5级以上地震极少，也不代表非非强震，可随意建设。本图意图更在于对地震带的基站的关注，加强保护与运维，保障通信安全。
### 参考
+ 通信基站地震破坏等级划分探讨 
+ 2016中国地震台网统一地震数据
+ 2016年中国基站数据


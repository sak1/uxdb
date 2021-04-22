

# UXDB实例

## UXDB在云环境下的部署

### 云环境下的UXDB是如何工作的

云环境下的UXDB分为基于DFS（分布式存储）和非DFS模式的。可以是基于OLTP模型的也可以是MPP模型的部署。UXDB在云环境下进行工作，需要云提供商提供支持，云服务提供商通常提供IaaS和PaaS服务。
在IaaS模式下，UXDB数据库将以VM的image模板方式发布到云环境中的Marketplace（Google称之为Catalog）的resource模板中，以便使用者进行资源分配。然后在分配好的资源上数据库管理员需要手动进行数据库配置。
在PaaS模式下，UXDB将以容器模式进行发布，如Docker容器模式，并以Service（服务）的模式发布在PaaS的可选服务中，当用户申请UXDB服务的时候，PaaS首先将调用IaaS层的功能完成基准VM的部署工作，然后执行PaaS中的服务执行脚本或程序对UXDB数据库进行适当的操作（如：“安装”、“升级”、“执行增加或减少节点”等操作）用以完成UXDB的配置，集群初始化以及启动等工作。当服务完成之后，UXDB将对外提供服务。
无论是IaaS和PaaS，运行态的UXDB将基本保持一致。鉴于UXDB是组件化的、可配置的数据库系统，为不同的需要，其部署和配置也将存在差异，下面的章节将介绍在云环境下各种不同的搭配组合的部署方式：

### 2.1	基于非DFS的一写多读方式的云端部署

应用场景：OLTP场景下的读写分离的应用场景

第一个部署在云环境下的UXDB数据库将成为master数据库，其后部署的数据库将采用replica的模式，对master数据库进行实时复制。具体架构如下：

 ![c1](D:\github\sak1\uxdb\img\c1.png)

当数据库集群通过云服务商初始化完成之后，在Cloud集群管理器中将记录该集群的信息，包括IP，端口以及缺省数据库等信息，并将用户名和密码发送到申请集群是设置的email。该集群初始化完成之后，信息将显示到WebAdmin上，用户可以通过WebAdmin对集群进行管理（启动，停止，监控等）。授权的用户可以通过WebAdmin管理创建数据库实例，并使用数据库。
如果云服务商提供了存储管理，并将存储与应用分开，那么当master数据库故障的时候，切换过程如下图所示：

![c2](D:\github\sak1\uxdb\img\c2.png)

 

如果云服务商没有提供存储管理，但仍然想要获取该效果，那么需要采用UXDB的DFS方式进行部署。

### 基于DFS的一写多读方式的云端部署

应用场景：OLTP场景下的读写分离的应用场景
该模式是在云环境不提供独立的存储模式的情况下，需要采用的模型。该模型是将数据库和存储分开的模型，使用DFS存储模型代替云环境的存储或者为云环境提供无分布式存储的不足，其他与2.1章节描述一致。具体架构如下：

![c3](D:\github\sak1\uxdb\img\c3.png)



与云提供商管理存储方式不同，DFS模式下的DFS将承担分布式存储的工作，数据库还可保持一写多读模式，这与1.1章节中的部署方式一致，仅需要对xlog的存放地点进行设置即可，对于DB Engine 1数据库，xlog需要放到DFS上，但对于其他数据库引擎（DB Engine 2、3），xlog应放到本地，避免replication的时候产生冲突。

### 基于多写多读方式的云端部署

应用场景：OLTP场景下的读写不分离的应用场景
多写多读需要采用共享存储模式，如果云服务商提供共享存储，其架构如下：

![c4](D:\github\sak1\uxdb\img\c4.png)


由于多写多读模型，要求所有数据库处理节点均为Master节点，且每个节点均可以进行读写操作，因此，不能将数据分散在多个节点上，应采用共享存储方式，该方式依赖云提供商提供统一的共享存储服务。如果云服务商不能提供共享存储服务，那么将采用DFS存储方式进行，其架构如下：

![c5](D:\github\sak1\uxdb\img\c5.png)

### 基于MPP的云端部署

应用场景：OLAP场景
与OLTP不同，OLAP需要将数据和分片和采用无共享架构，因此在云环境下的架构如下：

 ![c6](D:\github\sak1\uxdb\img\c6.png)

### 将单机模型下的UXDB数据库迁移到UXDB云环境

借助UXDB提供的迁移工具进行全数据库的迁移可以将单机环境下的数据库中的数据，表，试图，存储过程，触发器等全部无损的迁移到UXDB云环境中。应用程序只需要切换连接数据库时候的IP和端口即可，程序本身无影响。

### 为适应云环境，UXDB相应的适配工作

云服务商确定之后，UXDB需要结合云服务提供商的自身使用方式，进行一些响应的适配工作，该工作包括：制作合适的VM image以及容器 image，修改license以配合VM或者容器；开发一些列和云服务商相结合的脚本用于IaaS或者PaaS的集成。
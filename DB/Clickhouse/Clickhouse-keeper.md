Clickhouse-keeper是clickhouse在`21.8`版本后，提供的用于在clickhouse集群中，替代zookeeper的组件，其使用C++编写，并且使用RAFT架构。相比于zookeeper，其对写场景更加友好。

Clickhouse-keeper被作为clickhouse-server发行包的一部分，所以可以在clickhouse中增加配置，来激活此项服务。当然也可以作为独立的服务进行部署。

> 需要注意的是，clickhouse-keeper用于替代zookeeper服务，但仍然需要手动配置相关的集群配置，其与使用zookeeper是基本一致的。参考[官方配置](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication/)

# 部署方式

## 独立服务
Clickhouse-keeper可以作为独立的服务部署，如此部署，其与独立的zookeeper集群是类似的。

## 混合部署
Clickhouse-keeper可以与Clickhouse-server部署在一起，其会作为Clickhouse-server的内嵌服务运作，我们可以选择在复数个Clickhouse-server中添加对应的配置来激活这个服务。

> 混合部署时，我们可以在部分节点激活Clickhouse-keeper服务，而无需在所有节点上启动这个服务。


# 主要配置
keeper的配置位于`config.xml`中。
## keeper_server
keeper_server整体上分为三部分：
- 基础配置：配置端口、文件路径等基础信息。
- coordination_settings：高级配置。
- raft_configuration：集群配置。
### 基础配置

| 配置| 说明| m

### coordination_settings
### raft_configuration



## zookeeper



# 迁移
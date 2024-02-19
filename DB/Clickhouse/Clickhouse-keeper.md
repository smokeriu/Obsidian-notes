Clickhouse-keeper是clickhouse在`21.8`版本后，提供的用于在clickhouse集群中，替代zookeeper的组件，其使用C++编写，并且使用RAFT架构。相比于zookeeper，其对写场景更加友好。

Clickhouse-keeper被作为clickhouse-server发行包的一部分，所以可以在clickhouse中增加配置，来激活此项服务。当然也可以作为独立的服务进行部署。

> 需要注意的是，clickhouse-keeper用于替代zookeeper服务，但仍然需要手动配置相关的集群配置，其与使用zookeeper是基本一致的。参考[官方配置](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication/)

# 部署方式




# 主要配置

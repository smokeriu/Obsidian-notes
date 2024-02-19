Clickhouse-keeper是clickhouse在`21.8`版本后，提供的用于在clickhouse集群中，替代zookeeper的组件，其使用C++编写，并且使用RAFT架构。

Clickhouse-keeper被作为clickhouse-server发行包的一部分，所以可以在clickhouse中增加配置，来激活此项服务。当然也可以作为独立的服务进行部署。

> 需要注意的是，clickhouse-keeper用于替代zookeeper服务，但仍然需要手动配置相关的集群配置，其与使用zookeeper是基本一致的。参考[官方配置](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication/)

# 部署方式

## 独立服务
Clickhouse-keeper可以作为独立的服务部署，如此部署，其与独立的zookeeper集群是类似的。此时其关系类似：
![[assets/Pasted image 20240219150759.png]]


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

> 如果当前节点不作为keeper，则无需配置keeper_server。
### 基础配置

| 配置                   | 说明                                  | 默认值 |
| ---------------------- | ------------------------------------- | ------ |
| tcp_port               | 客户端(clickhouse-server)连接的端口。 | 2181，可考虑使用9181   |
| server_id              | keeper的server_id，必须是独立的数字。 |        |
| log_storage_path       | 数据存放路径。                        |        |
| snapshot_storage_path  | 快照存放路径。                        |        |
| enable_reconfiguration | 是否运行重新配置。                    | False  |

### coordination_settings
一些高级配置，可参考官方文档。
### raft_configuration
用于配置raft集群，基础配置仅仅是将该节点加入到keeper中，这里需要完整配置其他keeper节点。
raft_configuration下包含了复数个server节点，其配置了服务的具体信息：

| 配置              | 说明                                     |
| ----------------- | ---------------------------------------- |
| id                | 上文中配置的server_id                    |
| hostname          | 域名或ip                                 |
| port              | 集群内部的通信端口，需要与tcp_port不一致。 |
| can_become_leader | 能够成为leader                           |
需要注意的是，这里配置的port是raft集群内部的通信端口，仅用于此处。一个完整的raft_configuration如下：
```xml
<raft_configuration>
	<server>  
		<id>1</id>  
		<hostname>zoo1</hostname>  
		<port>9234</port>  
	</server>
	<server>
		<id>2</id>
		<hostname>zoo2</hostname>  
		<port>9234</port>
	</server>
	<server>
		<id>3</id>
		<hostname>zoo3</hostname>  
		<port>9234</port>
	</server>
</raft_configuration>
```
上述配置中，部署了一个三节点的keeper集群。我们需要在三个节点同时拥有这份配置。

一个完整的keeper_server配置如下：
```xml
<keeper_server>
	<tcp_port>9181</tcp_port>  
	<server_id>1</server_id>  
	<log_storage_path>.../clickhouse/log</log_storage_path>  
	<snapshot_storage_path>.../clickhouse/snapshots</snapshot_storage_path>
  
	<coordination_settings>
		<operation_timeout_ms>10000</operation_timeout_ms>
		<session_timeout_ms>30000</session_timeout_ms>
	</coordination_settings>

	<raft_configuration>
		<!-- server -->
	</raft_configuration>
</keeper_server>  
```
## zookeeper
由于keeper_server是一个相对独立的服务，我们需要通过配置zookeeper相关的配置，才能使集群脸上keeper。参考：[[Clickhouse的集群部署]]和[cluster-deployment](https://clickhouse.com/docs/en/architecture/cluster-deployment)

```xml
<zookeeper>  
	<node>  
		<host>zoo1</host>  
		<port>9181</port>  
	</node>  
	<node>  
		<host>zoo2</host>  
		<port>9181</port>  
	</node>  
	<node>  
		<host>zoo3</host>  
		<port>9181</port>  
	</node>  
</zookeeper>
```

# 修改配置
与zookeeper类似，clickhouse-keeper提供了动态配置的功能，可以通过命令来添加、移除keeper节点。

如果没有启用`enable_reconfiguration`，则需要手动配置并重启。

# 启动
如果采用内嵌的方式，则clickhouse-keeper会跟随clickhouse-server一起启动。如果需要单独启动，则：
```shell
clickhouse-keeper --config /etc/your_path_to_config/config.xml
```
某些时候，如果没有`clickhouse-keeper`命令，则使用：
```shell
clickhouse keeper --config /etc/your_path_to_config/config.xml
```

# 迁移
Clickhouse提供了工具，来将zookeeper中的数据迁移到Clickhouse-keeper中。有如下注意点：
1. 必须停止zookeeper。
2. zookeeper的版本需要高于3.4。

具体参考[migration-from-zookeeper](https://clickhouse.com/docs/en/guides/sre/keeper/clickhouse-keeper#migration-from-zookeeper)

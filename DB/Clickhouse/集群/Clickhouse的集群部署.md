部署Clickhouse集群前，需要拥有一个[[Clickhouse-keeper]]或zookeeper集群。并在集群中已经安装好clickhouse。

Clickhouse的集群机制是由自身配置决定的，并将其源数据存放与keeper集群中。


# 前置知识
在Clickhouse中，集群表是基于本地表的视图，集群表自身不存储数据，但可以将本地表理解为集群表的分片。具体内容参见[[Clickhouse的集群机制]]

# 配置

下述配置，除特殊说明，需要所有节点都配置，并保持配置一致。
## remote_servers

remote_servers用于定义一个集群：
```xml
<remote_servers>
	<cluster_name_1>
		<shard>...</shard>
		<shard>...</shard>
		...
	</cluster_name_1>
</remote_servers>
```

并配置这个集群的分片和副本信息：
```xml
<shard> 
    <weight>1</weight>
	<internal_replication>true</internal_replication>  
	<replica>  
		<host>chnode1</host>  
		<port>9000</port>  
	</replica>  
</shard>  
<shard>
	<weight>1</weight>
	<internal_replication>true</internal_replication>  
	<replica>  
		<host>chnode2</host>  
		<port>9000</port>  
	</replica>  
</shard>
```

其中：
- `shard`表示一个分片的配置。
- `weight`表示了这个分片的数据量权重。
- `replica`表示`shard`的一个副本配置。可以有多个replica。
- `replica.port`是`clickhouse-server`的tcp-port，默认为9000。
- `internal_replication`表示了副本复制的行为，当为true时，写入阶段只会写入其中一个副本，副本的数据复制由后续进程自动完成。为false时，则在写入节点就会写入所有副本。

上述配置创建了一个两个分片的集群，其中分片1会使用chnode1作为副本，而分片2则使用chnode2作为副本，总的来说，这个集群中，所有分片都是单副本的，如果需要为某个分片启用多副本，则在`shard`中配置多个`replica`项即可：

```xml
<shard>  
	<internal_replication>true</internal_replication>  
	<replica>  
		<host>chnode1</host>  
		<port>9000</port>  
	</replica>
	<replica>  
		<host>chnode2</host>  
		<port>9000</port>  
	</replica>
</shard>  
```

需要注意的是，如果clickhouse-server启用了加密，则配置replica时需要配置其账号密码：
```xml
<replica>  
	<host>chnode2</host>  
	<port>9000</port>
	<user>xxx</user>
	<password>xxx</password>
</replica>
```

## zookeeper
zookeeper用于配置keeper集群，使clickhouse-server能读写keeper集群：
```xml
<zookeeper>
	<node index="1">
		<host>chnode1</host>
		<port>9181</port>
	</node>
	<node index="2">
		<host>chnode2</host>
		<port>9181</port>
	</node>
	<node index="3">
		<host>chnode3</host>
		<port>9181</port>
	</node>
</zookeeper>
```

## macros
macros是一种宏配置，其用于进行连接替代。

需要保证不同clickhouse-server配置不同，取决于其属于的shard和replica配置。

```xml
<macros>  
	<shard>1</shard>  
	<replica>replica_1</replica>  
</macros>
```
经过配置后，在执行分布式命令时，分发到这个节点的命令中，`{shard}`会被替换为1，而`{replica}`会被替换为`replica_1`。

如有如下分布式SQL：
```sql
CREATE TABLE {table_name} ON CLUSTER {cluster_name}   
 (...)  
ENGINE = ReplicatedMergeTree('/path/to/tables/{shard}/table_name','{replica}')  
ORDER BY id;
```

则不同分片，不同副本的元数据，会在经过macors替换后，存放在不同的zookeeper路径下。实现元数据的隔离。
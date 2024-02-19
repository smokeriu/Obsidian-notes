部署Clickhouse集群前，需要拥有一个[[Clickhouse-keeper]]或zookeeper集群。并在集群中已经安装好clickhouse。

Clickhouse的集群机制是由自身配置决定的，并将其源数据存放与keeper集群中。


# 前置知识
在Clickhouse中，集群表是基于本地表的视图，集群表自身不存储数据，但可以将本地表理解为集群表的分片。具体内容参见[[Clickhouse的集群机制]]

# 配置

下述配置，除特殊说明，需要所有节点都配置。
## remote_servers

remote_servers用于定义一个集群，并配置这个集群的分片和副本信息：
```xml
<remote_servers>
	<cluster_name_1>
	  <!-- config for this cluster -->
	</cluster_name_1>
</remote_servers>
```

## 

# 集群表


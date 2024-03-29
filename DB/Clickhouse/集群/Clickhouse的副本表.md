以Replicated开头的表，称为副本表。其一般的创建格式为：
```sql
ENGINE = ReplicatedMergeTree('zoo_path','replica_name') 
```
其中：
- `zoo_path`表示其在keeper集群中的地址。
- `replica_name`表示这张表使用的是哪个副本。
由于clickhouse一般是一个集群，一般会采用分布式DDL来执行建表操作，这使得可以使用[[Clickhouse的集群部署#macros]]机制来由clickhouse只能替换：

```sql
CREATE TABLE ON CLUSTER <cluster_name>
...
ENGINE = ReplicatedMergeTree('/path/to/tables/{shard}/table_name','{replica}')
```

上述命令，会在集群中的所有机器上执行对应的命令，并替换相应的`shard`和`replica`。

需要注意的是，尽管在上述命令中创建replicated表时我们指定了`shard`，但单独的Replicated表并不能利用到分片特性，这主要是因为利用macro创建Replicated的方式主要是服务于[[Clickhouse的集群表]]，而Replicated本身并不需要在zoo_path中使用`{shard}`。
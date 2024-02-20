默认情况下，clickhouse的表只包含当前节点的数据，clickhouse的集群机制通过特殊的`Distributed Table Engine`实现。Distributed表是所有本地表的视图，其本身不存储数据。
# 创建

一般而言，创建分布式表前需要创建本地表，本地表一般约定使用`_local`作为后缀。其本地表一般会采用[[Clickhouse的副本表]]，从而可以同时利用分片和副本的性能。

> Distributed表使用的本地表并没有严格限制，一般约定上会使用副本表。

创建语句如下：
```sql
CREATE TABLE <table_name> [ON CLUSTER <cluster_name>] 
(...)
ENGINE = Distributed(cluster , database, table, [,sharding_key[, policy_name]])
```
其中：
- `table`表示分布式表所映射的本地表名。
- `sharding_key`是一个选填参数，可以翻译成分片键，也可以理解为分布键。
    - 在往分布式表中**插入**数据时，会按照`sharding_key`和`config.xml`中配置的分片权重(weight)一同决定写入分布式表时的路由, 即数据最终落到哪个物理表上。权重越大，写入其中的数据越多。
    - 可以是表中一列的原始数据(如`site_id`)，也可以是函数调用的结果, 如采用了随机值`rand()`。
    - 该键要尽量保证数据均匀分布，一个常用的操作是采用区分度较高的列的哈希值，如`intHash64(user_id)`。
- `plicy_name` - 用于决定中间数据存储在哪。一般需要配合[storage_configuration](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree/#table_engine-mergetree-multiple-volumes_configure)使用。

请注意，`sharding_key`是一个很重要的参数，其决定了向集群表中插入数据时如何进行分布。但由于`local`表和集群表本质上是相对独立的，所以`sharding_key`只是对数据分布的弱约束（例如其无法管理直接插入`local`表的数据分布）。这主要会影响到一些Join场景的使用。

# 查询





# 写入
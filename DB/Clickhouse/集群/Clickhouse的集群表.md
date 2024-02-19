默认情况下，clickhouse的表只包含当前节点的数据，clickhouse的集群机制通过特殊的`Distributed Table Engine`实现。Distributed表是本地表的视图，其本身不存储数据。
# 创建

一般而言，创建分布式表前需要创建本地表，本地表一般约定使用`_local`作为后缀。其本地表一般会采用

# 查询

# 写入
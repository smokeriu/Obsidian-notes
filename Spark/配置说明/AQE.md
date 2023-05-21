首先。
- AQE无法解决小文件，其更多的是为了解决数据倾斜的场景。
- AQE无法解决Range-Join，因为Range-Join最后卡在某个进程并不是数据倾斜导致的，而是Range-Join在Spark中会转换为**Broadcast nested loop join**。


# 主要配置
## Coalescing Post Shuffle Partitions
用于在shuffle时，合并发出的分区，注意，要使用该功能，需要启用AQE。
主要控制参数：
| 参数                                                      | 含义                                                                 | 默认值 |
| --------------------------------------------------------- | -------------------------------------------------------------------- | ------ |
| spark.sql.adaptive.coalescePartitions.enabled             | 是否开启该功能。                                                     | true   |
| spark.sql.adaptive.coalescePartitions.parallelismFirst    | 开启后将通过minPartitionSize来计算并最大化并行度，避免性能回退。     | true   |
| spark.sql.adaptive.coalescePartitions.minPartitionSize    | coalesce后最小的分区大小。不得高于advisoryPartitionSizeInBytes的20%  | 1MB    |
| spark.sql.adaptive.coalescePartitions.initialPartitionNum | coalesce前的初始分区数。如果不设置则使用`spark.sql.shuffle.partitions` | none   |
| spark.sql.adaptive.advisoryPartitionSizeInBytes   | 用于建议spark                                                                    |        |

## 赛奥

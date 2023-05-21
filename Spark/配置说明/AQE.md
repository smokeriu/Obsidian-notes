首先。
- AQE无法解决小文件，其更多的是为了解决数据倾斜的场景。
- AQE无法解决Range-Join，因为Range-Join最后卡在某个进程并不是数据倾斜导致的，而是Range-Join在Spark中会转换为**Broadcast nested loop join**。


# 主要配置

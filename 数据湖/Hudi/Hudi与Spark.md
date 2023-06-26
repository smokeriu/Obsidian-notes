# 初始化

初始化有几个注意点：
1. 需要使用hudi的catalog。
2. 需要使用hudi的extensions。
3. 需要使用`KryoSerializer`。
```java
config("spark.serializer", 
	   "org.apache.spark.serializer.KryoSerializer")  
config("spark.sql.catalog.hudi", 
	   "org.apache.spark.sql.hudi.catalog.HoodieCatalog")  
config("spark.sql.extensions",
	   "org.apache.spark.sql.hudi.HoodieSparkSessionExtension")
```

# 批任务

## 读
Hudi读取数据基于文件存放的位置。
```scala
val tripsSnapshotDF = spark.read
	.format("hudi")
	.load(basePath)
```

### 历史快照
Hudi也支持根据时间读取历史快照
```scala
// 注意这个不是时间戳
option("as.of.instant", "20210728141108100")
option("as.of.instant", "2021-07-28 14:11:08.200")
// equal to 2021-07-28 00:00:00
option("as.of.instant", "2021-07-28")
```
即读取这个时间点，hudi中的数据快照。

### 增量读取
Hudi支持增量读取，其含义为：读取指定时间范围内更新/插入的**最新**数据。
与历史快照的主要区别是：
- 得到的数据永远是最新的数据。
- 只会得到时间范围内变化的数据。
在增量读取模式下，需要提供如下参数：
- `hoodie.datasource.query.type`：读取方式，设定为`incremental`。
- `hoodie.datasource.read.begin.instanttime`：范围起始时间
	- 精确到秒：`20230626181800`。注意并非时间戳。
- `hoodie.datasource.read.end.instanttime`：范围终止时间。
	- 可不提供，则使用当前时间。


## 写

# 流任务
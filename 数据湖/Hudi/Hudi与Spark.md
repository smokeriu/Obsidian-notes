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
option("as.of.instant", "20210728141108100")
option("as.of.instant", "2021-07-28 14:11:08.200")
// equal to 2021-07-28 00:00:00
option("as.of.instant", "2021-07-28")
```
即读取这个时间点，hudi中的数据快照。

### 增量读取
Hudi支持增量读取，其含义为：读取指定时间fj'ww'l


## 写

# 流任务
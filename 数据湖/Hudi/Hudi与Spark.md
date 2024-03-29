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
读取hudi，除了自身的数据外，还会将hudi的元数据读取出来。

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
> 历史快照模式是默认模式，所以可以不指定`hoodie.datasource.query.type`。如果没有设置参数`as.of.instant`，则其实就是读取当前时间的快照。

特别的，如果指定的快照时间早于第一次写入，则会读取到空数据（带schema）。

### 读取优化模式
另一种模式类似于历史快照的方式。与历史快照的区别在于，他只会读取已经合并的文件，这使得其相比于默认的快照读取拥有更高的延迟，但能够减少查询的性能消耗（只限于merge_on_read类型的表）。
- `hoodie.datasource.query.type`：读取方式，设定为`read_optimized`。

和历史快照一样，这种模式也可以通过指定`as.of.instant`来读取某一个时刻的快照。

> 1. 如果表模式是merge_on_read，则使用该模式可能会无法看到部分实际已经写入的数据（这些数据还在log文件中等待合并）。由于hudi没有一个常驻服务，一般是等下次写入时，由hudi判断是否值得合并。
> 2. 如果表模式是copy_on_write，则这个参数与快照的意义是一样的。

### 增量读取
Hudi支持增量读取，其含义为：读取指定时间范围内更新/插入的**最新**数据。
与历史快照的主要区别是：
- 得到的数据永远是**最新**的数据。
	- 如果记录1在时刻B被修改，时刻C又被修改，则无论指定何种范围，查出来的都是时刻C被修改的后的数据。
- 只会得到时间**范围内发生变化**的数据。
	- 如果记录1在时刻B被修改，记录2在时刻C被修改，如果范围只包含时刻C，则只会查到记录2。


在增量读取模式下，需要提供如下参数：
- `hoodie.datasource.query.type`：读取方式，设定为`incremental`。
- `hoodie.datasource.read.begin.instanttime`：范围起始时间。必填。
	- 精确到秒：`20230626181800`。注意并非时间戳。
- `hoodie.datasource.read.end.instanttime`：范围终止时间。
	- 可不提供，则使用当前时间。



## 写
Hudi在写入时，有两层来控制写入逻辑：
1. 由Spark的SaveMode来控制初始逻辑。
2. 有Hudi的OPERATION_OPT_KEY来控制实际逻辑。

### Spark SaveMode
- Overwrite：
- Append：


# 流任务
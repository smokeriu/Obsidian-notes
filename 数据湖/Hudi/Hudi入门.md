# 项目地址
[Apache Hudi](https://hudi.apache.org/cn/)
> 0.13.1版本

# 基本概念
## 元数据字段
Hudi现阶段**必须需要**3种元数据字段用于对数据进行去重和分区。
### 主键和分区
在hudi中，主键和分区是有关联性的。hudi的每个记录都由HoodieKey唯一标识，而HoodieKey由主键和分区键共同组成。
- `RECORDKEY_FIELD_OPT_KEY`：配置哪些字段用于生成主键。默认为`uuid`。
- `PARTITIONPATH_FIELD_OPT_KEY`：配置哪些字段用于生成分区键。默认为`partitionpath`。

默认（布隆索引）情况下，主键只保证其在分区下的唯一性。HoodieKey保证全局唯一性。
主键和分区建都是hudi的**必要**字段。

通过`KEYGENERATOR_CLASS_OPT_KEY`可以指定如何生成键（主键和分区键）：
- `SimpleKeyGenerator`：即建由单个字段构成。
- `ComplexKeyGenerator`（默认）：即建由多个字段组成的元组构成。也可以只指定单个字段。
- `TimestampBasedKeyGenerator`：是单字段类型的一种实现，会将分区字段转换为timestamp类型而非string类型。主键则不受影响。
- `CustomKeyGenerator`：即将所有数据写入同一个分区的情况，此时的分区需要填入空字符串。
- `GlobalDeleteKeyGenerator`：即hoodieKey经由主键构成，是数据全局唯一。其仅适用于包含批量删除的逻辑中。且需要搭配全局索引一起使用。

> 主键和分区建的生成方式目前需要保持一致，即如果使用单字段，则主键和分区都只能指定单个字段。

#### TimestampBasedKeyGenerator
这种建生成器是SimpleKeyGenerator的一种实现，其特点是：
- 分区需要使用时间字段，且会被转换为timestamp类型，而非string类型。
- 主键仍然使用指定的列，但只能是单个列。
其需要提供如下参数：
- `hoodie.deltastreamer.keygen.timebased.timestamp.type`：类型。
	- 即指定为分区的字段格式，包括了：
		- `UNIX_TIMESTAMP`：字段就是一个时间戳的情况。如果输入字段是其他类型，则是秒类型的`SCALAR`。
		- `DATE_STRING`：字符串的日期，需要指定输入格式。
		- `MIXED`：混合模式，不过官方没有说明，看代码与DATE_STRING用途一致。
		- `EPOCHMILLISECONDS`：毫秒级的`SCALAR`类型。
		- `SCALAR`：根据指定的`time.unit`，来生成时间， 是`EPOCHMILLISECONDS`的拓展版。
	- 如果输入的数据为null，则会转换为1970的初始时间。
- `hoodie.deltastreamer.keygen.timebased.input.dateformat`：输入格式。
- `hoodie.deltastreamer.keygen.timebased.output.dateformat`：输出格式。
	- 格式都是形如：`yyyy-MM-dd`。
- `hoodie.deltastreamer.keygen.timebased.timezone`：时区。
	- 如：`UTC`，`GMT+8:00`等。
- `hoodie.deltastreamer.keygen.timebased.timestamp.scalar.time.unit`。标量的单位
	- 当type类型为`SCALAR`时，指定这个配置来决定scalar的单位，如days，hours等。

总的而言，如果输入的是字符串类型，则需要指定`input.dateformat`。而`timestamp.type`更多的则是控制一些附加参数，如`UNIX_TIMESTAMP`隐含了`scalar.time.unit`为秒。

#### 其它参数
有一些其它参数，用于控制分区建的表现形式：
- `hoodie.datasource.write.partitionpath.urlencode`：是否编码url样式的分区。默认`false`。
	- 如果不编码，而分区字段中存在如`/`的数据，则会产生子文件夹。
- `hoodie.datasource.write.hive_style_partitioning`：分区是否采用hive样式。默认`false`。
	- 即生成`key=value`的文件夹样式，而不是直接使用value。

### 合并
由于Hudi具有唯一标识，当具有同样HoodieKey的数据写入时（或者是同一批数据中存在同样的数据时），Hudi根据预合并来决定如何保留数据。预合并需要指定预合并字段：
- `PRECOMBINE_FIELD_OPT_KEY`：选择一个字段，参与合并比较。默认为`ts`。
- `WRITE_PAYLOAD_CLASS`：决定了如何进行合并比较。

这个合并过程大体分为如下几个阶段：
1. 预合并。预合并会先从所有待写入数据中筛选出实际写入的数据。
	1. 当同一批次中存在重复HoodieKey的数据时，如何选择使用哪条数据。
		1. 这里的同一批次应该指的是一个Job的记录，而非某个分区或某次commit的。
		2. Spark中使用了`Rdd.reduce`来进行预合并。
		3. 参考：`table.action.commit.HoodieWriteHelper#doDeduplicateRecords`。
	2. 如果写入逻辑为insert时，预合并默认是关闭的。
		1. 取决与参数：`hoodie.combine.before.insert`。
2. 合并。比较存储中的数据和预合并得到的待插入数据。
	1. 当存储中已经存在重复HoodieKey的数据时，如何选择使用哪条数据。

Hudi内置了数个合并比较逻辑：
- `OverwriteWithLatestAvroPayload`（默认）：永远使用最新数据。
- `DefaultHoodieRecordPayload`：
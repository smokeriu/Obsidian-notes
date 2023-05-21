一般而言，我们会使用coalesce来合并分区，避免使用repartition，因为后者会产生shuffle。
不过coalesce会将合并分区尽可能下推，从而影响shuffle的并行度。

如下两种情况需要使用repartition而非coalesce：

# 避免coalesce
## 没有shuffle，但包含复杂的UDF
在一般认识下，coalesce会经由窄依赖进行下推，如果使用的某个function是耗时的，这会导致并行度降低，从而影响性能。

## 包含了shuffle，但收缩过于剧烈
coalesce的用于将分区进行合并，但如果指定的参数收缩得过于剧烈（如设置为1），这回导致Shuffle计算也被集中到单个节点上。

# 如何选择

缩小分区优先使用coalesce，但需要注意，如果分区收缩过于剧烈，则应当使用repartition。



参考:
- [coalesce-reduces-parallelism-of-entire-stage-spark](https://stackoverflow.com/questions/44494656/coalesce-reduces-parallelism-of-entire-stage-spark)
- [spark-coalesce20-overwrite-parallelism-of-repartition1000-groupbyxxx-apply](https://stackoverflow.com/questions/57952905/spark-coalesce20-overwrite-parallelism-of-repartition1000-groupbyxxx-apply)
- [spark-repartition-vs-coalesce](https://stackoverflow.com/questions/31610971/spark-repartition-vs-coalesce)
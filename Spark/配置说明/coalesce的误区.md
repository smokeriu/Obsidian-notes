一般而言，我们会使用coalesce来合并分区，避免使用repartition，因为后者会产生shuffle。
不过coalesce会将合并分区尽可能下推，从而影响shuffle的并行度。

在一般认识下，coalesce会经由窄依赖进行下推，如果使用的某个function是耗时的，这会导致并行度降低，从而影响性能。

参考:
- [coalesce-reduces-parallelism-of-entire-stage-spark](https://stackoverflow.com/questions/44494656/coalesce-reduces-parallelism-of-entire-stage-spark)
- [spark-coalesce20-overwrite-parallelism-of-repartition1000-groupbyxxx-apply](https://stackoverflow.com/questions/57952905/spark-coalesce20-overwrite-parallelism-of-repartition1000-groupbyxxx-apply)
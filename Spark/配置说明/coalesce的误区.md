一般而言，我们会使用coalesce来合并分区，避免使用repartition，因为后者会产生shuffle。
不过coalesce会将合并分区尽可能下推，从而影响shuffle的并行度：

参考:
- []()
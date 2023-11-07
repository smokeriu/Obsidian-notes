
Spark在访问带有kerberos认证的hbase时，需要提供参数：
- `spark.security.credentials.hbase.enabled`并设置为true。

> 在旧版本中为`spark.yarn.security.credentials.hbase.enabled`。

# 参数用途

Spark通过HadoopDelegationTokenProvider接口来进行统一的认证管理，实现上包含了Hadoop，Hive，Kafka，Hbase，Es等。

当`spark.security.credentials.hbase.enabled`设置为true时，
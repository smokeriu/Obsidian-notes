
Spark在访问带有kerberos认证的hbase时，需要提供参数：
- `spark.security.credentials.hbase.enabled`并设置为true。

> 在旧版本中为`spark.yarn.security.credentials.hbase.enabled`。

# 参数用途

Spark通过`org.apache.spark.security.HadoopDelegationTokenProvider`接口来进行统一的认证管理，实现上包含了Hadoop，Hive，Kafka，Hbase，Es等。

Spark启动时，会通过`HadoopDelegationTokenManager#isServiceEnabled`来判断是否启用了对应服务的tokenManager，其格式为：`spark.security.credentials.%s.enabled`。所以当`spark.security.credentials.hbase.enabled`设置为true时，会将`HBaseDelegationTokenProvider`加入到管理中，用于管理Hbase的Token。

在程序运行中，Spark会调用`obtainDelegationTokens`方法来获取票据授权，具体则是使用`Provider`的`obtainDelegationTokens`方法。

# 坑



# 追踪

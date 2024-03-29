
> Spark version 3.3.2

Spark在访问带有kerberos认证的hbase时，需要提供参数：
- `spark.security.credentials.hbase.enabled`并设置为true。

> 在旧版本中为`spark.yarn.security.credentials.hbase.enabled`。

# 参数用途

Spark通过`org.apache.spark.security.HadoopDelegationTokenProvider`接口来进行统一的认证管理，实现上包含了Hadoop，Hive，Kafka，Hbase，Es等。

Spark启动时，会通过`HadoopDelegationTokenManager#isServiceEnabled`来判断是否启用了对应服务的tokenManager，其格式为：`spark.security.credentials.%s.enabled`。所以当`spark.security.credentials.hbase.enabled`设置为true时，会将`HBaseDelegationTokenProvider`加入到管理中，用于管理Hbase的Token。

在程序运行中，Spark会调用`obtainDelegationTokens`方法来获取票据授权，具体则是使用`Provider`的`obtainDelegationTokens`方法。方法会进行票据授权，并返回有效时间。

# 坑

通过`HBaseDelegationTokenProvider`的代码，可以发现实际使用的是`org.apache.hadoop.hbase.security.token.TokenUtil`的`obtainToken`方法通过`hadoopConf`获取token，并且写到说这个方法在hbase-2.0后就被移除了，所以还提供了另一种途径来获取token：先获取Hbase的Connection，再调用`obtainToken`的参数为Connection的方法。
```scala
val obtainTokenMethod = mirror.classLoader  
  .loadClass("org.apache.hadoop.hbase.security.token.TokenUtil")  
  .getMethod("obtainToken", connectionParamTypeClassRef)
```

不过实际上，`TokenUtil`这个类在较新的Hbase-client包中被替换为了`ClientTokenUtil`，不过Spark这里并没有跟进，并且当加载不到类时，只是使用warn进行提示，不容易发现问题：
```scala
logWarning(Utils.createFailedToGetTokenMessage(serviceName, e))
```


# 追踪

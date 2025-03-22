# Kerberos
1. 安装


# HDFS
## DataNode
DataNode要启用kerberos，需要额外的方式，这里采用SASL的方式：

1. 额外添加配置：

```xml
<property>
                <name>dfs.http.policy</name>
                <value>HTTPS_ONLY</value>
</property>
<property>
                <name>dfs.datanode.address</name>
                <value>0.0.0.0:61004</value>
        </property>
        <property>
                <name>dfs.datanode.http.address</name>
                <value>0.0.0.0:61006</value>
        </property>

        <property>
                <name>dfs.data.transfer.protection</name>
                <value>integrity</value>
</property>
```
# Yarn
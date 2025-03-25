
> 以Ubuntu为例。

# 特殊说明

这个集群主要用于测试，尽可能减少部署复杂度。有如下特殊说明：
1. kerberos用户采用与linux用户名称一致，如ssiu/hadoop。

# 前置条件

## 设置免密

1. 生成key：

```shell
ssh-keygen -t rsa
```

2.  向目标服务器复制key，这里是本地

```shell
ssh-copy-id user@host
```

## Kerberos
1. 安装kerberos：

```shell
sudo apt install krb5-admin-server, krb5-user -y
```

这个过程如果是交互式，会设置realm等信息，这里假设设置realm为：`HADOOP.COM`。

2. 配置kerberos数据库（注意域的一致性）：

```shell
sudo kdb5_util  -r HADOOP.COM create -s
```

3. 启动kerberos（不同系统可能有差异）：

```shell
sudo service krb5-admin-server start
sudo service krb5-kdc start
```

4. 进入kadmin：

```shell
sudo kadmin.local
```

5. 添加用户，并导出keytab：

```shell
addprinc admin/admin
addprinc hdfs/hadoop
addprinc yarn/hadoop

ktadd -k /path/to/hdfs.keytab hdfs/hadoop
ktadd -k /path/to/yarn.keytab yarn/hadoop
```

需要注意的是，由于进入kadmin.local使用的sudo，所以导出完keytab后，还需要修改所有人，使当前用户可以读取keytab文件。

另外，这里部署的是单机版，所以instance直接设置的机器域名：`hadoop`，对于集群，可以使用：`hdfs/_HOST`。

# HDFS

1. 配置`core-site.xml`：

```xml
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop:9000</value>
</property>

    <!-- 开启安全验证 -->
<property>
    <name>hadoop.security.authentication</name>
    <value>kerberos</value>
</property>
<property>
    <name>hadoop.security.authorization</name>
    <value>true</value>
</property>
<property>
    <name>hadoop.tmp.dir</name>
    <value>/home/ssiu/data/tmp</value>
</property>
```

> 请确保：`${hadoop.tmp.dir}`指向的文件夹存在。


2. 配置`hdfs-site.xml`：

```xml
<property>
        <name>dfs.replication</name>
        <value>1</value>
</property>
<property>
        <name>dfs.namenode.kerberos.principal</name>
        <value>hdfs/hadoop@HADOOP.COM</value>
</property>
<property>
        <name>dfs.namenode.keytab.file</name>
        <value>/home/ssiu/app/hadoop/etc/hadoop/hdfs.keytab</value>
</property>
<property>
        <name>dfs.datanode.kerberos.principal</name>
        <value>hdfs/hadoop@HADOOP.COM</value>
</property>
<property>
        <name>dfs.datanode.keytab.file</name>
        <value>/home/ssiu/app/hadoop/etc/hadoop/hdfs.keytab</value>
</property>
<property>
        <name>dfs.secondary.namenode.kerberos.principal</name>
        <value>hdfs/hadoop@HADOOP.COM</value>
</property>
<property>
        <name>dfs.secondary.namenode.keytab.file</name>
        <value>/home/ssiu/app/hadoop/etc/hadoop/hdfs.keytab</value>
</property>
<property>
        <name>dfs.block.access.token.enable</name>
        <value>true</value>
</property>
        
<property>
        <name>dfs.name.dir</name>
        <value>/home/ssiu/data/name</value>
</property>
<property>
        <name>dfs.data.dir</name>
        <value>/home/ssiu/data/data</value>
</property>
```

> 请确保：`${dfs.name.dir}`和`${dfs.data.dir}`指向的文件夹存在。
> 请确保`principal`和`keytab`配置正确。

3. 配置`workers`：将本机的hostname添加到`workers`文件中。
## DataNode
DataNode要启用kerberos，需要额外的方式，这里采用SASL的方式：

1. 额外添加配置到`hdfs-site.xml`。

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

2. 生成SSL

通过下述语句生成SSL相关配置和文件，可以在一个目录统一管理，例如：`/home/ssiu/ssl`。下述过程需要设置密码，可以设置为统一密码，方便后续配置。

```shell
openssl req -new -x509 -keyout bd_ca_key -out bd_ca_cert -days 9999 -subj '/C=CN/ST=beijing/L=beijing/O=test/OU=test/CN=test'

keytool -keystore keystore -alias localhost -validity 9999 -genkey -keyalg RSA -keysize 2048 -dname "CN=test, OU=test, O=test, L=beijing, ST=beijing, C=CN"

keytool -keystore truststore -alias CARoot -import -file bd_ca_cert

keytool -certreq -alias localhost -keystore keystore -file cert

openssl x509 -req -CA bd_ca_cert -CAkey bd_ca_key -in cert -out cert_signed -days 9999 -CAcreateserial -passin pass:123456

keytool -keystore keystore -alias CARoot -import -file bd_ca_cert

keytool -keystore keystore -alias localhost -import -file cert_signed
```

3. 配置SSL信息

进入hadoop配置目录，通过模版文件得到：`ssl-server.xml`和`ssl-client.xml`。需要修改的配置如下：

- `ssl-server.xml`：

```xml
<property>
  <name>ssl.server.truststore.location</name>
  <value>/home/ssiu/ssl/truststore</value>
  <description>Truststore to be used by NN and DN. Must be specified.
  </description>
</property>

<property>
  <name>ssl.server.truststore.password</name>
  <value>123456</value>
  <description>Optional. Default value is "".
  </description>
</property>

<property>
  <name>ssl.server.keystore.location</name>
  <value>/home/ssiu/ssl/keystore</value>
  <description>Keystore to be used by NN and DN. Must be specified.
  </description>
</property>

<property>
  <name>ssl.server.keystore.password</name>
  <value>123456</value>
  <description>Must be specified.
  </description>
</property>

<property>
  <name>ssl.server.keystore.keypassword</name>
  <value>123456</value>
  <description>Must be specified.
  </description>
</property>
```

- `ssl-client.xml`：

```xml
<property>
  <name>ssl.client.truststore.location</name>
  <value>/home/ssiu/ssl/truststore</value>
  <description>Truststore to be used by clients like distcp. Must be
  specified.
  </description>
</property>

<property>
  <name>ssl.client.truststore.password</name>
  <value>123456</value>
  <description>Optional. Default value is "".
  </description>
</property>

<property>
  <name>ssl.client.keystore.location</name>
  <value>/home/ssiu/ssl/keystore</value>
  <description>Keystore to be used by clients like distcp. Must be
  specified.
  </description>
</property>

<property>
  <name>ssl.client.keystore.password</name>
  <value>123456</value>
  <description>Optional. Default value is "".
  </description>
</property>

<property>
  <name>ssl.client.keystore.keypassword</name>
  <value>123456</value>
  <description>Optional. Default value is "".
  </description>
</property>
```

## 启动

1. 需要首先格式化hdfs：

```shell
hdfs namenode -format
```

2. 启动服务

```shell
sbin/start-dfs.sh
```

# Yarn

1. 配置`yarn-site.xml`：

```xml
<property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
</property>
<property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>hadoop:8030</value>
</property>
<property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>true</value>
</property>
<property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
</property>
<property>
        <name>yarn.resourcemanager.principal</name>
        <value>yarn/hadoop@HADOOP.COM</value>
</property>
<property>
        <name>yarn.resourcemanager.keytab</name>
        <value>/home/ssiu/app/hadoop/etc/hadoop/yarn.keytab</value>
</property>
<property>
        <name>yarn.nodemanager.principal</name>
        <value>yarn/hadoop@HADOOP.COM</value>
</property>
<property>
        <name>yarn.nodemanager.keytab</name>
        <value>/home/ssiu/app/hadoop/etc/hadoop/yarn.keytab</value>
</property>
<property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>4096</value>
</property>
```


2. 启动yarn：

```shell
sbin/start-yarn.sh
```
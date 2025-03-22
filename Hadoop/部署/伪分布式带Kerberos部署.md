
# 前置条件

## 设置免密

1. 

## Kerberos
1. 安装


# HDFS
## DataNode
DataNode要启用kerberos，需要额外的方式，这里采用SASL的方式：

1. 额外添加配置

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
openssl req -new -x509 -keyout test_ca_key -out test_ca_cert -days 9999 -subj '/C=CN/ST=China/L=Beijin/O=hduser/OU=security/CN=hadoop.com'

keytool -keystore keystore -alias localhost -validity 9999 -genkey -keyalg RSA -keysize 2048 -dname "CN=hadoop.com, OU=test, O=test, L=Beijing, ST=China, C=cn"

openssl x509 -req -CA test_ca_cert -CAkey test_ca_key -in cert -out cert_signed -days 9999 -CAcreateserial -passin pass:123456

keytool -keystore keystore -alias CARoot -import -file test_ca_cert

keytool -keystore keystore -alias localhost -import -file cert_signed
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
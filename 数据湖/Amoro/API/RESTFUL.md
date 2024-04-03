> 需要使用http-server的地址，一般为1630端口
# 登陆

地址：`ams/v1/login`
方式：`POST`
Body：`JSON`

```json
{
	"user": "admin",
	"password": "admin"
}
```

## Java代码
```java
// 设置cookie
CookieStore cookieStore = new BasicCookieStore();
CloseableHttpClient httpClient = HttpClients  
        .custom()  
        .setDefaultCookieStore(cookieStore)  
        .build();

final HttpPost post = new HttpPost("http://127.0.0.1:1630/ams/v1/login");  
String body = "{\"user\": \"admin\", \"password\": \"admin\"}";  
final StringEntity entity = new StringEntity(body, ContentType.APPLICATION_JSON);  
post.setEntity(entity);  
httpClient.execute(post);
```

# 获取catalog

> 获取catalog前，需要先登陆获取cookie

地址：`ams/v1/catalogs`
方式：`GET`

## Java代码

```java
final HttpGet get = new HttpGet("http://127.0.0.1:1630/ams/v1/catalogs");  
final CloseableHttpResponse execute = httpClient.execute(get);
```

响应：
```json
{
	"message": "success",
	"code": 200,
	"result": [
		{
			"catalogName": "test1",
			"catalogType": "ams",
			"storageConfigs": {
				"hive.site": "xxx",
				"hadoop.hdfs.site": "xxx",
				"hadoop.core.site": "xxx",
				"storage.type": "Hadoop"
			},
			"authConfigs": {
				"auth.kerberos.krb5": "xxx",
				"auth.type": "kerberos",
				"auth.kerberos.keytab": "xxx",
				"auth.kerberos.principal": "kyuubi/admin@CDH.COM"
			},
			"catalogProperties": {
				"table.self-optimizing.group": "OP1",
				"warehouse": "hdfs:///amoro/local",
				"uri": "http://127.0.0.1:1630/api/iceberg/rest",
				"catalog-impl": "org.apache.iceberg.rest.RESTCatalog",
				"table-formats": "MIXED_ICEBERG"
			},
			"setCatalogName": true,
			"setCatalogType": true,
			"storageConfigsSize": 4,
			"setStorageConfigs": true,
			"authConfigsSize": 4,
			"setAuthConfigs": true,
			"catalogPropertiesSize": 5,
			"setCatalogProperties": true
		}
	]
}
```

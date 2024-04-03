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

> 获取catalog填，
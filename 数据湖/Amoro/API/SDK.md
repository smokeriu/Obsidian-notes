通过SDK可以进行：创建数据库、创建表、获取数据库、获取表等操作。但catalog相关操作目前只能通过[[RESTFUL]]完成。

# Maven

> Maven尚不在公共库，需要install或者deploy到私有库

```xml
<dependency>  
    <groupId>com.netease.amoro</groupId>  
    <artifactId>amoro-core</artifactId>  
    <version>${amoro.version</version>  
</dependency>
```
通过SDK可以进行：创建数据库、创建表、获取数据库、获取表等操作。但catalog相关操作目前只能通过[[RESTFUL]]完成。


# Maven

> Maven尚不在公共库，需要install或者deploy到私有库。

> 本文对应的版本为`0.7-SNAPSHOT`。

```xml
<dependency>  
    <groupId>com.netease.amoro</groupId>  
    <artifactId>amoro-core</artifactId>  
    <version>${amoro.version}</version>  
</dependency>
```

# 获取Catalog
所有操作需要先获取Catalog实例：

```java

```
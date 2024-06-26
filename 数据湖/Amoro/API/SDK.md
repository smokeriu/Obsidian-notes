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

# 获取ArcticCatalog
所有操作需要先获取Catalog实例：

```java
final ArcticCatalog catalog = CatalogLoader.load(url);
```

其中url可以使用`thrift`接口，形如：`thrift://127.0.0.1:1260/catalog_name`。

# 数据库

## 创建数据库
ArcticCatalog提供了方法来创建数据库：
```java
void createDatabase(String databaseName);
```

> 如果数据库已经存在，则会抛出：`AlreadyExistsException`。
## 获取数据库列表
ArcticCatalog提供了方法来列出所有数据库。
```java
List<String> listDatabases();
```
# 表

## 创建表
创建表需要构建Schema和TableIdentifier：
Schema
```java
final Schema schema = new Schema(  
        Types.NestedField.required(1, "id", Types.IntegerType.get()),  
        Types.NestedField.optional(2, "name", Types.StringType.get()),  
        Types.NestedField.required(3, "ts", Types.TimestampType.withoutZone())  
);
```

TableIdentifier：
```java
final TableIdentifier ti = TableIdentifier.of("test1", "testdb", "tb1"),  
```

紧接着构建TableBuilder：
```java
final TableBuilder builder = catalog.newTableBuilder(  
        ti,
        schema  
);
```

在TableBuilder中，可以配置主键、分区等信息：
```java
builder.withPrimaryKeySpec(  
        PrimaryKeySpec.builderFor(schema)  
                .addColumn("id")  
                .build()  
).withPartitionSpec(  
        PartitionSpec.builderFor(schema)  
                .day("ts", "ts_day")  
                .build()  
)
```

最后通过builder的create方法，即可完成创建。
## 获取表列表
ArcticCatalog提供了方法来列出数据库下的所有表。
```java
List<TableIdentifier> listTables(String database);
```

> 返回的TableIdentifier中包含了catalog，database等信息。
## 获取表详情
ArcticCatalog提供了方法来获取表的详细信息。
```java
ArcticTable loadTable(TableIdentifier tableIdentifier);
```

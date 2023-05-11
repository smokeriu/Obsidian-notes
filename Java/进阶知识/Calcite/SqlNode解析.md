使用上，需要先获取`SqlParser.Config`。
```java
private static final SqlParser.Config DEFAULT_CONFIG = SqlParser.config()  
	.withParserFactory(SqlParseImpl.FACTORY)  
	.withQuotedCasing(Casing.UNCHANGED)  
	.withUnquotedCasing(Casing.UNCHANGED);
```

其中，SqlParseImpl为SQL解析实现，后续实现自定义的语句解析时，也是实现这个。
在获取了`SqlParser.Config`后，便需要获取parse实例，并可解析获得sqlNode。
```java
final SqlParser sqlParser = SqlParser.create(sql, config);
final SqlNode sqlNode = sqlParser.parseStmt();
```

sqlNode是一个顶级父类，比较常用到的有：
- SqlCall：一个Sql操作，如Select，Join，Where等。
- SqlLiteral：一个常量。
- SqlIdentifier：一个标识符，如表名，字段名等。
- SqlNodeList：由sqlNode组成的集合，如select


# 常用类
## SelectCall
表示一条Select字句。其定义主要有：
```java
SqlNodeList keywordList;  
SqlNodeList selectList;  
@Nullable SqlNode from;  
@Nullable SqlNode where;  
@Nullable SqlNodeList groupBy;  
@Nullable SqlNode having;  
SqlNodeList windowDecls;  
@Nullable SqlNodeList orderBy;  
@Nullable SqlNode offset;  
@Nullable SqlNode fetch;  
@Nullable SqlNodeList hints;
```
都对应一个select会包含的元素。
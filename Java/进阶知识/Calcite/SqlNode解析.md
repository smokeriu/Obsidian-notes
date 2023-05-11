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
- SqlCall
- SqlLiteral：一个常量。
- SqlIdentifier：一个标识符，
- SqlNodeList：由sqlNode组成的集合，一般用于描述一个配置集合，如`Create Table`的列定义就是一个SqlNodeList
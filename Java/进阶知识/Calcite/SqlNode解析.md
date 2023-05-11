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

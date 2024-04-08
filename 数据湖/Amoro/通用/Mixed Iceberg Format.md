相比于iceberg格式，提供如下功能：
1. 主键。
2. 更好的OLAP性能。
3. LogStore带来的低延迟。
4. 更好的事务处理。

# 架构
![[assets/Pasted image 20240401160249.png]]
- BaseStore：由批计算、优化程序产生的数据文件。
- ChangeStore：存储表中变化的数据，记录由流计算、Update处理等产生的改动。
- LogStore：作为 ChangeStore 的缓存层，可加快流处理速度。

在实际的API定义中，一般分为了两种表类型：
- BaseTable：基础数据。
- ChangeTable：存储修改数据的Table。

因此包含了共五类数据：
```java
public enum DataFileType {  
  BASE_FILE(0, "B"),  
  INSERT_FILE(1, "I"),  
  EQ_DELETE_FILE(2, "ED"),  
  POS_DELETE_FILE(3, "PD"),  
  ICEBERG_EQ_DELETE_FILE(4, "IED");
}
```

- BASE_FILE：存储在BaseTable中的记录的文件。
- INSERT_FILE：存储在ChangeTable中的记录的文件。
- EQ_DELETE_FILE：存储在ChangeTable中删除记录的文件。
- POS_DELETE_FILE：存储在BaseTable中删除记录的文件。
- ICEBERG_EQ_DELETE_FILE：原生ICEBERG中Equi
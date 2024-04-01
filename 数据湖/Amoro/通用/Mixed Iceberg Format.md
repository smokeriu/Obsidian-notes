相比于iceberg格式，提供如下功能：
1. 主键。
2. 更好的OLAP性能。
3. LogStore带来的低延迟。
4. 更好的事务处理。

# 架构
![[assets/Pasted image 20240401160249.png]]
- BaseStore：由批计算、优化程序产生的数据文件。
- ChangeStore：由流计算、Update处理等产生的

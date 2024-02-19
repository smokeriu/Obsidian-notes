以Replicated开头的表，称为副本表。其创建格式为：
```sql
ENGINE = ReplicatedMergeTree('/path/to/tables/{shard}/table_name','{replica}') 
```
其中：
- `zoo_path`表示其在keeper集群中的地址。
- replica_namebu n
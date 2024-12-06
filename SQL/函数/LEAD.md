#窗口函数 

用于获取当前行之后的行数据，与[[LAG]]用法一致。

用于获取**下一条**数据的时间。
```sql
lead(time, 1) OVER (PARTITION BY uid ORDER BY time) pre_time
```

需要注意，这里的下一条，取决于`ORDER BY`中的顺序。
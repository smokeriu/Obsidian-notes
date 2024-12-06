#窗口函数

用于返回之前行的元素，需要搭配窗口函数使用。


下述SQL返回同一个用户的，**上一条**记录中的time字段。

```sql
lag(time, 1) OVER (PARTITION BY uid ORDER BY time) pre_time
```

需要注意，这里的上一条，取决于`ORDER BY`中的顺序。
用于返回之前行的元素，需要搭配窗口函数使用。


下述SQL返回同一个用户的，上一条ji l

```sql
lag(time, 1) OVER (PARTITION BY uid ORDER BY time desc) pre_time
```

Mysql中并没有Split，explode函数，可以通过其他途径实现某些功能。

这里假设列表字段为String，并用csv存储

# 查找元素个数

我们可以通过计算replace前后的长度，来间接计算其长度。

```sql
length(tag) - length(replace(tag, ',', '')) + 1
```

即分隔符被溢出后，减少的数量，可以看作`元素数量 - 1`。
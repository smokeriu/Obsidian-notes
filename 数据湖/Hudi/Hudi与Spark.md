# 初始化

初始化有几个注意点：
1. 需要使用hudi的catalog。
2. 需要使用hudi的extensions。
3. 需要使用`KryoSerializer`。
```java
config("spark.serializer", 
	   "org.apache.spark.serializer.KryoSerializer")  
config("spark.sql.catalog.hudi", 
	   "org.apache.spark.sql.hudi.catalog.HoodieCatalog")  
config("spark.sql.extensions", "org.apache.spark.sql.hudi.HoodieSparkSessionExtension")
```

# 批任务

## 读



## 写

# 流任务
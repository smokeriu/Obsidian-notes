pandas提供了get_dummies函数，用于将非数值数据转换为数值数据。本质利用的是列转行的逻辑，即将这一列的数据转换为对应的列，根据值将其对应的位置转换为0或1：
![[assets/Pasted image 20230810144258.png]]
> 这图可能有一点小问题，pandas会在新列名中，将原有列名作为前缀加上，即新的三列为：`Color_Red`，`Color_Yellow`，`Color_Green`。
# 定义
```python
def get_dummies(data, dummy_na=False, drop_first=False , ...)
```
其中，
- `data`：表示需要处理的数据，一般是DataFrame。
- `dummy_na`：增加一列来表示NaN值，即将原有数据中的NaN值形成单独一列。
	- 默认为false，则会丢弃所有NaN的特征值。
- `drop_first`：是否丢弃第一列。
	- 因为转成列后，其实我们依靠`k-1`列已经能够描述所有数据。所以某些情况下我们可以丢弃第一列。

默认情况下，会对data中的*所有*非数值列应用独热编码，**数值列则不会编码**。


# drop_first
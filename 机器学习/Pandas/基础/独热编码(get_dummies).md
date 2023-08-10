pandas提供了get_dummies函数，用于将非数值数据转换为数值数据。本质利用的是列转行的逻辑，即将这一列的数据转换为对应的列，根据值将其对应的位置转换为0或1：
![[assets/Pasted image 20230810144258.png]]
# 定义
```python
def get_dummies(data, dummy_na=False, drop_first=False , ...)
```
其中，
- `data`：表示需要处理的数据，一般是DataFrame。
- `dummy_na`：
- `drop_first`：是否丢弃第一列。
	- 因为转成列后，

默认情况下，会对data中的所有非数值列应用独热编码。
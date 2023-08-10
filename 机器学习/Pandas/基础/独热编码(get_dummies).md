pandas提供了get_dummies函数，用于将非数值数据转换为数值数据。本质利用的是列转行的逻辑，即将这一列的数据转换为对应的列，根据值将其对应的位置转换为0或1：
![[assets/Pasted image 20230810144258.png]]
> 这图可能有一点小问题，pandas会在新列名中，将原有列名作为前缀加上，即新的三列为：`Color_Red`，`Color_Yellow`，`Color_Green`。
# 定义
```python
def get_dummies(data, columns=None, dummy_na=False, drop_first=False, ...)
```
其中，
- `data`：表示需要处理的数据，一般是DataFrame。
- `columns`：表示要处理的列名列表。默认为None，表示所有`object`列。
	- 指定columns可以强制对非`object`列进行编码。
- `dummy_na`：增加一列来表示NaN值，即将原有数据中的NaN值形成单独一列。
	- 默认为false，则会丢弃所有NaN的特征值。
- `drop_first`：是否丢弃第一列。
	- 因为转成列后，其实我们依靠`k-1`列已经能够描述所有数据。所以某些情况下我们可以丢弃第一列。

默认情况下，会对data中的*所有*非数值列应用独热编码，**数值列则不会编码**。
> 更严格来说，是对object列进行编码。

# 示例
```python
pd.get_dummies(df) # 对所有object类型的列应用独热编码。
pd.get_dummies(df, columns=['A','B']) # 只对A列和B列应用独热编码。
```

# 注意
需要避免对离散值特别多的列应用独热编码，例如某一列的离散值有100k个，那么会得到100k列。

# 独热编码在机器学习中的陷阱
在预处理机器学习数据时，直接使用hget_dummies会导致虚拟变量陷阱，即我们产出了多列特征，但这些特征之间其实是冗余的，这被称为**多重共线性**。
某些情况我们需要破坏**多重共线性**，但不绝对。
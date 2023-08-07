Torch提供了一些生成张量的方法。

# 向量
## arange
```python
torch.arange(start=0, end, step=1, out=None, dtype=None, requires_grad=False) -> Tensor
```
生成一个左闭右开的向量，其中：
- start：起始值。
- end：终止值。（不包含）
- step：步长。
- out：结果张量。用于将生成结果复制给变量。
	- 如果指定了out，方法返回值和out对应的变量，会拥有一样的数据。
- dtype：类型
- requires_grad：是否需要计算梯度。

> 由于后三个参数基本都会有，后面就不解释了。

## range
与arange类似，不过生成的是左闭右闭的向量。

# 张量
## rand
**均匀分布**
```python
torch.rand(*sizes, out=None) -> Tensor
```

返回一个张量，包含了从区间`[0, 1)`的**均匀分布**中抽取的一组随机数。张量的形状由参数sizes定义。
- size：为一个整数序列。定义了张量的形状。

例如：
```python
torch.rand(2,3)
# 生成一个2*3的矩阵
```
# randn
**标准正态分布**
```python
torch.rand(*sizes, out=None) -> Tensor
``` 

返回一个张量，包含了从**标准正态分布**（均值为0，方差为1，即高斯白噪声）中抽取的一组随机数。张量的形状由参数sizes定义。

# normal
**离散正态分布**
```python
torch.normal(means, std, out=None) -> Tensor
```
返回一个张量，包含了均值为`means`，方差为`std`的离散正态分布。张量的形状由参数means和std共同定义。

- means：是一个**张量**，包含每个输出元素相关的正态分布的均值。
- std：是一个**张量**，包含每个输出元素相关的正态分布的标准差。
均值和标准差的形状不须匹配，但每个张量的**元素个数须相同**。

例如：
```python

```
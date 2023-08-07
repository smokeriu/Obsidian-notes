Torch提供了两个函数来生成随机分布的数据
# rand
**均匀分布**
```python
torch.rand(*sizes, out=None) -> Tensor
```

返回一个张量，包含了从区间`[0, 1)`的**均匀分布**中抽取的一组随机数。张量的形状由参数sizes定义。
- size：为一个整数序列。定义了张量的形状。
- out：结果张量。用于将生成结果复制给变量。
	- 如果指定了out，方法返回值和out对应的变量，会拥有一样的数据

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
返回一个张量，包含了均值为means，方差为std的离散正态分布。
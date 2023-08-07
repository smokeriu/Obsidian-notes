Torch提供了一些生成张量的方法。

# 向量
## arange
```python
torch.arange(start=0, end, step=1, out=None, dtype=None, requires_grad=False) -> Tensor
```
返回一个一维张量，生成一个左闭右开的向量，其中：
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

## linspace
**线性间距向量**
```python
torch.arange(start, end, steps)
```
返回一个一维张量，包含在区间start和end上均匀间隔的**step个点**。这里steps指定的*是个数，而不是步长*。
## logspace
```python
torch.logspace(start, end, steps, base=10.0)
```
返回一个一维张量，同样是生成steps个点，但需要指定对数的底（base）。


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
torch.normal(means, std, size, out=None) -> Tensor
```
返回一个张量，包含了均值为`means`，方差为`std`的离散正态分布。张量的形状由参数means和std共同定义。
- means：是一个**张量**，包含每个输出元素相关的正态分布的均值。
	- 也可以是一个float类型的标量，此时所有元素均使用该值作为均值。
- std：是一个**张量**，包含每个输出元素相关的正态分布的标准差。
	- 也可以是一个float类型的标量，此时所有元素均使用该值作为标准差。
特别的，如果means和std都为张量，则需要形状yi'vi

例如：
```python
means = torch.arange(0,10)
std = torch.linspace(1,0,10)
torch.normal(means,std)
# 因为means是一维张量，所以结果也是一维张量
```

特别的，如果means和std都是float，则需要指定size。
例如：
```python
torch.normal(means,std, [2,3])
# 生成一个
```
Torch提供了clamp函数，用于将数据进行强制收缩。
定义：
```python
torch.clamp(input, min, max, out=None) → Tensor
```
其中：
- input：待收缩的tensor。
- min：收缩的下限。
- max：收缩的上限。

这里的收缩是硬性的，即：
```python
x = x if min <= x <= max
x = min if x < min
x = max if x > max
```

# 示例
我们可以将jp'go
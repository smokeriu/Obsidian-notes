Torch提供了两个函数来生成随机分布的数据
# rand

```python
torch.rand(*sizes, out=None) -> Tensor
```

返回一个张量，包含了从区间`[0, 1)`的均匀分布中抽取的一组随机数。张量的形状由参数sizes定义。
- size：为一个整数序列。定义了张量的形状。
- out：结果张量。

例如：
# randn

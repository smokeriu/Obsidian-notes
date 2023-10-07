torch中，张量拥有repeat方法，用于在指定的维度上*repeat*多次：
```python
a = torch.randn(33, 55)

a.repeat(1,1) # 33 * 55
a.repeat(2,1) # 66 * 55
```

特别的，如果repeat参数的维度超出了张量的维度，则相当于：
```python

```
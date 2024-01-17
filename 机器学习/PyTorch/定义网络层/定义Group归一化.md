```python
nn.GroupNorm(num_groups: int, 
			 num_channels: int, 
			 eps: float = 1e-5, 
			 affine: bool = True,...)
```
其中：
- `num_groups`：即分组数。
- `num_channels`：即特征/通道数量。
- `eps`：分母的固定常数，避免分母为0。
- `affine`：是否需要仿射。即是否学习$\gamma$和$\beta$。

# 样例

PyTorch提供了内置的[[../../深度学习v2/基础知识/归一化/GroupNormalization|GroupNormalization]]。
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
> `num_groups`需要能整除`num_channels`。

特别的，GroupNorm要求的InputShape是`[N,C,*]`，所以我们需要permute。
# 样例

```python
# 假定有如下输入：
output_from_pre_layer = torch.randn(size=[8, 224, 224, 32])

# 定义LN
gn = nn.GroupNorm(4, 32)
out = gn(output_from_pre_layer.permute(0,3,1,2)) # [8, 32, 224, 244]
```
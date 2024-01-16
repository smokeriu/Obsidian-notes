InstanceNormalization在nn包下，与[[定义批量归一化]]类似，针对不同类型的输入数据也分为了InstanceNorm2d，InstanceNorm3d等。且参数含义也基本一致：
```python
nn.InstanceNorm2d(num_features: Int,
				  eps=1e-05,
				  momentum=0.1,
				  affine=True,
				  track_running_stats=True)
```
其实：
- `num_features`：即特征/通道数量。
- `eps`：分母的固定常数，避免分母为0。
- `momentum`：运行过程调整移动平均的参数。
- `affine`：是否需要仿射。即是否学习$\gamma$和$\beta$。
- `track_running_stats`：是否跟踪数据来更新预测时使用的均值和方差。
与批量归一化一样，输入的数据需要将维度变换为`(N, C, H, W)`。

# 样例
```python

```
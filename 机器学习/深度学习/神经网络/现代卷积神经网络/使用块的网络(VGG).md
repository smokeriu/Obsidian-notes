# VGG块
经典卷积神经网络的基本组成部分是下面的这个序列：
1. 带填充以保持分辨率的卷积层。
2. 非线性激活函数，如ReLU。
3. 汇聚层，如最大汇聚层。
而一个VGG块与之类似，由**一系列卷积层**组成，后面再加上用于空间下采样的最大汇聚层。

用PyTorch表示一个VGG块：
```python
import torch
from torch import nn
from d2l import torch as d2l

def vgg_block(num_convs, in_channels, out_channels):
    layers = []
    for _ in range(num_convs):
        layers.append(nn.Conv2d(in_channels, out_channels,
                                kernel_size=3, padding=1))
        layers.append(nn.ReLU())
        in_channels = out_channels
    layers.append(nn.MaxPool2d(kernel_size=2,stride=2))
    return nn.Sequential(*layers)
```
其中：
- `num_convs`：卷积层的数量。
- `in_channels`：输入通道的数量。
- `out_channels`：输出通道数量。
通过观察，我们发现，
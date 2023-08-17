# VGG块
经典卷积神经网络的基本组成部分是下面的这个序列：
1. 带填充以保持分辨率的卷积层。
2. 非线性激活函数，如ReLU。
3. 汇聚层，如最大汇聚层。
而一个VGG[[../../学习计算/块|块]]与之类似，由**一系列卷积层**组成，后面再加上用于空间下采样的最大汇聚层。

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
通过观察，我们发现其特点是：
- 多个累积的卷积层串联。
- 从第二个卷积层开始，输入通道数就等于输出通道数了。
- 最后通过一个汇聚层采样，而不是每个卷积层后都跟随一个汇聚层。
用图像表示为：
![[assets/Pasted image 20230817105155.png|300]]

## VGG网络
整个VGG网络的结构如图所示：
![[assets/Pasted image 20230817105257.png|200]]
除了重复使用VGG块以外，其实网络整体与[[深度卷积神经网络(AlexNet)]]是类似的，都是先经过多个[[../卷积神经网络/卷积层|卷积层]]，再通过[[../../多层感知机/多层感知机|多层感知机]]将数据平铺。

原始VGG网络有5个卷积块，其中前两个块各有一个卷积层，后三个块各包含两个卷积层。 第一个模块有64个输出通道，每个后续模块将输出通道数量翻倍，直到该数字达到512。由于该网络使用8个卷积层和3个全连接层，因此它通常被称为VGG-11。

VGG-11可以通过PyTorch表示：
```python
conv_arch = ((1, 64), (1, 128), (2, 256), (2, 512), (2, 512))
def vgg(conv_arch):
    conv_blks = []
    in_channels = 1
    # 卷积层部分
    for (num_convs, out_channels) in conv_arch:
        conv_blks.append(vgg_block(num_convs, in_channels, out_channels))
        in_channels = out_channels

    return nn.Sequential(
        *conv_blks, nn.Flatten(),
        # 全连接层部分
        nn.Linear(out_channels * 7 * 7, 4096), nn.ReLU(), nn.Dropout(0.5),
        nn.Linear(4096, 4096), nn.ReLU(), nn.Dropout(0.5),
        nn.Linear(4096, 10))

net = vgg(conv_arch)
```
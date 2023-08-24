[[../卷积神经网络/卷积层|卷积层]]的输入和输出由*四维张量*组成，张量的每个轴分别对应样本、通道、高度和宽度。而全连接层的输入和输出通常是分别对应于样本和特征的二维张量。

在[[../卷积神经网络/LeNet|LeNet]]、[[深度卷积神经网络(AlexNet)]]、[[使用块的网络(VGG)]]中，最后都是将卷积层压平后，通过全连接层输出输出，这会导致参数的急剧膨胀。
- 对于一个卷积层，其参数为：$输入通道数 \times 输出通道数 \times 卷积窗口大小$。
- 对于一个全连接层，则其参数为：$输入通道数 \times 卷积数据分辨率 \times 全连接分辨率$。
这使得全连接层的参数可以轻松突破百万。

NiN的思路是，替代掉全连接层，具体想法是，如果我们将权重连接到每个空间位置，我们可以将其视为[1x1卷积层](1x1卷积层.md)，从另一个角度看，即将空间维度中的**每个像素视为单个样本**，将**通道维度视为不同特征**。

NiN块以一个普通卷积层开始，后面是两个1×1的卷积层。这两个1×1卷积层充当带有ReLU激活函数的逐像素全连接层。

用图展示：
![[assets/Pasted image 20230824151008.png|200]]

其中，每一个卷积块之间使用最大[[../卷积神经网络/汇聚层|汇聚层]]相连，最终的结果则使用平均汇聚层。
每个卷积块用PyTorch表示为：
```python
def nin_block(in_channels, out_channels, kernel_size, strides, padding):
    return nn.Sequential(
        nn.Conv2d(in_channels, out_channels, kernel_size, strides, padding),
        nn.ReLU(),
        nn.Conv2d(out_channels, out_channels, kernel_size=1), nn.ReLU(),
        nn.Conv2d(out_channels, out_channels, kernel_size=1), nn.ReLU())
```
可以看到，后面接了两个1x1的卷积层，且输入输出通道数相等。
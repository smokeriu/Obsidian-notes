> 是最早发布的卷积神经网络之一

总体来看，LeNet（LeNet-5）由两个部分组成：
- 卷积编码器：由两个卷积层组成。
- 全连接层密集块：由三个全连接层组成。
其结构如下图所示：
![[assets/Pasted image 20230816171802.png]]
每个卷积块中的基本单元是一个卷积层、一个sigmoid激活函数和平均汇聚层。
> 虽然ReLU和最大汇聚层更有效，但它们在20世纪90年代还没有出现。

用PyTorch表示，则为：
```python
net = nn.Sequential(
	nn.Conv2d(1, 6, kernel_size=5, padding=2), nn.Sigmoid(),
	nn.AvgPool2d(kernel_size=2, stride=2),
	nn.Conv2d(6, 16, kernel_size=5), nn.Sigmoid(),
	nn.AvgPool2d(kernel_size=2, stride=2),
	nn.Flatten(),
	nn.Linear(16 * 5 * 5, 120), nn.Sigmoid(),
	nn.Linear(120, 84), nn.Sigmoid(),
	nn.Linear(84, 10))
```
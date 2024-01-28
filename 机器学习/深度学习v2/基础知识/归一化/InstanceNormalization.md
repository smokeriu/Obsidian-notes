Instance Normalization(IN)在Channel上进行归一化，适合用于GAN(生成式神经网络)。

IN在计算归一化统计量时并没有像BN那样跨样本、单通道，也没有像LN那样单样本、跨通道。它是取的单通道，单样本上的数据进行计算。
$$
\begin{aligned}
& \mu_{ti}=\frac1{HW}\sum_{l=1}^W\sum_{m=1}^Hx_{tilm} \\
& \sigma_{ti}^2=\frac1{HW}\sum_{l=1}^W\sum_{m=1}^H(x_{tilm}-\mu_{ti})^2 \\
\end{aligned}
$$
其与BN和LN的差异如图所示：
![[assets/Pasted image 20240116190656.png]]
由于IN在`[H,W]`上进行归一化，显然均值结果是一个矩阵。

# PyTorch
PyTorch预置了IN，参见[[../../../PyTorch/定义网络层/定义Instance归一化|定义Instance归一化]]。


# Why and When
对于一些图像风格迁移（IST，Image Style Transfer），BN和LN都不适合。

- BN，每个样本的每个像素点的信息都是非常重要的，而BN计算归一化统计量时考虑了一个批量中所有图片的内容，从而造成了每个样本独特细节的丢失。
- LN，LN则会忽略不同通道的差异，也不适合这种情况。

由于IN只在单通道和单样本上进行归一化，适用于**批量较小且单独考虑每个像素点**的场景中。在图像这类应用中，每个通道上的值是比较大的，因此也能够取得比较合适的归一化统计量。但如下场景中，因为无法基于单通道和单样本得到最够的数据，并不适用使用IN：
- MLP或者RNN中：因为在MLP或者RNN中，每个通道上只有一个数据，这时会自然不能使用IN。
- Feature Map比较小时：因为此时IN的采样数据非常少，得到的归一化统计量将不再具有代表性。
# 参考
- [模型优化之Instance Normalization](https://zhuanlan.zhihu.com/p/56542480)
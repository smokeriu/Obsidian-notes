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


# PyTorch


# 参考
- [模型优化之Instance Normalization](https://zhuanlan.zhihu.com/p/56542480)
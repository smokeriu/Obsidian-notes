Instance Normalization(IN)在Channel上进行归一化，适合用于GAN(生成式神经网络)。

IN在计算归一化统计量时并没有像BN那样跨样本、单通道，也没有像LN那样单样本、跨通道。它是取的单通道，单样本上的数据进行计算。

归一化的目的就是使得预处理的数据被限定在一定的范围内（比如`[0,1]`或者`[-1,1]`），从而消除奇异样本数据导致的不良影响。
在[[数据预处理]]阶段，就可以对数据进行归一化。也可以用在处理输入前。

常见的归一化公式有：
- 最大最小标准化。
- Z-score标准化。
- log对数函数归一化。
- 反正切函数归一化。
- 范数归一化。


不过这里讨论的是归一化层。即对上一层产生的数据进行归一化后，再传递到下一层。

神经层的参数变化会导致其输入的分布发生较大的差异。利用随机梯度下降更新参数时，每次参数更新都会导致网络中间每一层的输入的分布发生改变。越深的层，其输入分布会改变的越明显。上述过程称为**内部协变量偏移**。归一化层就是用来缓解上述问题。
常见的归一化层有：
1. [[归一化/BatchNormalization|BatchNormalization]]。
2. [LayerNormalization](LayerNormalization.md)。
3. [InstanceNormalization](InstanceNormalization.md)。
4. [GroupNormalization](GroupNormalization.md)。


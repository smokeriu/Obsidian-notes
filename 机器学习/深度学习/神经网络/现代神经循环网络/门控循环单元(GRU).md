门控循环单元，与普通的循环神经网络之间的关键区别在于：前者支持隐状态的门控。 这意味着模型有专门的机制来确定应该*何时更新隐状态*， 以及应该*何时重置隐状态*。

# 重置门
重置门允许我们控制“可能还想记住”的过去*状态的数量*。

设定重置门为$\mathbf{R}_t \in \mathbb{R}^{n \times h}$，则有：
$$
\mathbf{R}_t = \sigma(\mathbf{X}_t \mathbf{W}_{xr} + \mathbf{H}_{t-1} \mathbf{W}_{hr} + \mathbf{b}_r)
$$
其中，时间步为$t$：
- 输入为$\mathbf{X}_t \in \mathbb{R}^{n \times d}$。
- 上一个时间步的隐状态为$\mathbf{H}_{t-1} \in \mathbb{R}^{n \times h}$。
- 激活函数表示为$\sigma$。
- 权重为$\mathbf{W}_{xr}\in \mathbb{R}^{d \times h}$和$\mathbf{W}_{hr} \in \mathbb{R}^{h \times h}$。
- 偏置为$\mathbf{b}_r \in \mathbb{R}^{1 \times h}$。

显然由于偏置的形状，与之求和时会触发广播机制。

# 更新门
更新门将允许我们控制新状态中*有多少个*是旧状态的副本。

设定更新门为$\mathbf{Z}_t \in \mathbb{R}^{n \times h}$，则有：
$$
\mathbf{Z}_t = \sigma(\mathbf{X}_t \mathbf{W}_{xz} + \mathbf{H}_{t-1} \mathbf{W}_{hz} + \mathbf{b}_z)
$$
整体上，其形状如图：
![[assets/Pasted image 20230919195828.png|500]]
# 候选隐状态
将重置门与常规隐状态更新机制集成，得到时间步$t$的候选隐状态$\tilde{\mathbf{H}}_t \in \mathbb{R}^{n \times h}$：
$$
\tilde{\mathbf{H}}_t = \tanh(\mathbf{X}_t \mathbf{W}_{xh} + \left(\mathbf{R}_t \odot \mathbf{H}_{t-1}\right) \mathbf{W}_{hh} + \mathbf{b}_h),
$$
其中：
- 权重参数为$\mathbf{W}_{xh} \in \mathbb{R}^{d \times h}$和$\mathbf{W}_{hh} \in \mathbb{R}^{h \times h}$。
这里使用[[../../../../数学/基础/向量与图形/Hadamard乘积|Hadamard乘积]]来处理重置门和上一步的隐状态，可以减少以往状态的影响。显然，$\mathbf{R}_t \$
特别的，当$\mathbf{R}_t$中的项接近1时，则取值全部依赖于$\mathbf{H}_{t-1}$，恢复为普通的循环神经网络，而如果
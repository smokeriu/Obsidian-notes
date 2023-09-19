门控循环单元，与普通的循环神经网络之间的关键区别在于：前者支持隐状态的门控。 这意味着模型有专门的机制来确定应该*何时更新隐状态*， 以及应该*何时重置隐状态*。

# 重置门
重置门允许我们控制“可能还想记住”的过去状态的数量。

设定重置门为$\mathbf{R}_t \in \mathbb{R}^{n \times h}$。则有：
$$
\mathbf{R}_t = \sigma(\mathbf{X}_t \mathbf{W}_{xr} + \mathbf{H}_{t-1} \mathbf{W}_{hr} + \mathbf{b}_r)
$$
其中，时间步为$t$：
- 输入为$\mathbf{X}_t \in \mathbb{R}^{n \times d}$。
- 上一个时间步的隐状态为$\mathbf{H}_{t-1} \in \mathbb{R}^{n \times h}$。

# 更新门
更新门将允许我们控制新状态中有多少个是旧状态的副本。
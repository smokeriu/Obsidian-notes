长期以来，隐变量模型存在着长期信息保存和短期输入缺失的问题。 解决这一问题的最早方法之一是长短期存储器。

# 门
与[[门控循环单元(GRU)]]类似，LSTM也是通过门的方式实现记忆存储的，LSTM总共设计了三种门：
- 遗忘门$\mathbf{F}_t$：用来重置单元的内容。
- 输出门$\mathbf{O}_t$：用来从单元中输出条目。
- 输入门$\mathbf{I}_t$：用来决定何时将数据读入单元。
其结构如图：
![[assets/Pasted image 20230921191024.png|500]]
其公式与[[../循环神经网络/循环神经网络#循环层|循环层]]基本一致，为带有[[../../多层感知机/激活函数#Sigmoid函数|Sigmoid激活函数]]*的全连接层*，如下：
$$
\begin{split}\begin{aligned}
\mathbf{I}_t &= \sigma(\mathbf{X}_t \mathbf{W}_{xi} + \mathbf{H}_{t-1} \mathbf{W}_{hi} + \mathbf{b}_i),\\
\mathbf{F}_t &= \sigma(\mathbf{X}_t \mathbf{W}_{xf} + \mathbf{H}_{t-1} \mathbf{W}_{hf} + \mathbf{b}_f),\\
\mathbf{O}_t &= \sigma(\mathbf{X}_t \mathbf{W}_{xo} + \mathbf{H}_{t-1} \mathbf{W}_{ho} + \mathbf{b}_o),
\end{aligned}\end{split}
$$
# 候选记忆元
在LSTM中，引入了名叫**记忆元**的要素，其与隐状态类似，会参与下一个时间步的计算。在引入记忆元前，先了解候选记忆元$\tilde{\mathbf{C}}_t$，其公式上与门类似，不过将激活函数更换成了[[../../多层感知机/激活函数#tanh函数|tanh]]，更接近于循环层：
$$
\tilde{\mathbf{C}}_t = \text{tanh}(\mathbf{X}_t \mathbf{W}_{xc} + \mathbf{H}_{t-1} \mathbf{W}_{hc} + \mathbf{b}_c)
$$
其结构如图：
![[assets/Pasted image 20230921192613.png|500]]

# 记忆元
输入门控制采用多少来自$\tilde{\mathbf{C}}_t$的新数据，而遗忘门控制保留多少过去的记忆元$\mathbf{C}_{t-1} \in \mathbb{R}^{n \times h}$。则我们可以得到时间步$t$的记忆元公式：
$$
\mathbf{C}_t = \mathbf{F}_t \odot \mathbf{C}_{t-1} + \mathbf{I}_t \odot \tilde{\mathbf{C}}_t
$$
简而言之：
- 如果遗忘门$\mathbf{F}_t$中的各项接近0，则当前步的记忆元$\mathbf{C}_t$将**遗忘**上一个时间步的记忆元$\mathbf{C}_{t-1}$，而主要取值于当前步的候选记忆元$\tilde{\mathbf{C}}_t$。
- 如果输入门$\mathbf{I}_t$中的各项接近0，则当前步的记忆元$\mathbf{C}_t$将不考虑当前步的候选记忆元$\tilde{\mathbf{C}}_t$，我们可以看做是*忽略*当前步的输入$\mathbf{X}_t$。

引入这种设计是为了缓解梯度消失问题， 并更好地捕获序列中的长距离依赖关系。

则整体流程如下所示：
![[assets/Pasted image 20230921194018.png|500]]
# 隐状态
目前，这一层只是产出了记忆元，显然还需要生成隐状态，这里会利用到输出门：
$$
\mathbf{H}_t = \mathbf{O}_t \odot \tanh(\mathbf{C}_t).
$$
它仅仅是记忆元的$tanh$的门控版本，确保了$\mathbf{H}_t$的值肯定在$(0,1)$之间。并且：
- 对于输出门接近1，我们就能够有效地将*所有记忆信息*传递给预测部分。
- 对于输出门接近0，我们只保留记忆元内的所有信息，而*不需要更新*隐状态（隐状态$\mathbf{H}_t$全为0）。


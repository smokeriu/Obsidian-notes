PyTorch可以通过`LSTM`函数定义[长短期记忆网络(LSTM)](长短期记忆网络(LSTM).md)层：
```python
nn.LSTM(input_size: int, 
		hidden_size: int, 
		num_layers: int = 1,
		bias: bool = True,
		batch_first: bool = False,
		dropout: int = 0,
		bidirectional: bool = False)
```
其中：
- `input_size`表示输入数据的大小（特征数）。
	- 假设输入是一个单词，则`input_size`表示的是单词向量的长度。参考[独热编码](机器学习/PyTorch/基础/独热编码.md)。
- `hidden_size`表示每个隐藏层中，节点的个数。
- `num_layers`表示隐藏层的层数，默认为1。
- `bias`默认为`True`，为`False`时，则表示不使用偏置参数。
- `batch_first`默认为`False`，为`True`时表示输入张量第一维是批量大小。
	- 一般而言，循环神经网络的第一维表示序列时间步，第二维才表示批量。
	- 即如果输入句子单词数为10（假设一个单词是一个输入向量），则第一维为10。
- `dropout`默认为0，若为非0，则表示除了最后一层，其它层都会插入[暂退层](定义暂退层.md)。
- `bidirectional`默认为Flase，为`True`表示使用双向LSTM。

使用时，输入为：`input, (h_0, c_0)`：
- `input`：输入张量，形状为`(seq_len, batch, input_size)`。
- `h_0`：即隐状态矩阵，形状为`(num_layers * num_directions, batch, hidden_size)`。
	- `num_directions`即方向数量，`bidirectional`为`True`是为2，否则为1。
- `c_0`：即神经元矩阵，形状为`(num_layers * num_directions, batch, hidden_size)`。

> 特别的，`(h_0,c_0)`为可选参数。

该层产生的输出为：`(output, h_n, c_n)`：
- `output`：输出张量，形状为`(seq_len, batch, num_directions * hidden_size)`。
- `h_n`：为最后一层的隐状态，形状为`(num_layers * num_directions, batch, hidden_size)`。
- `c_0`：即最后一层的神经元，形状为`(num_layers * num_directions, batch, hidden_size)`。


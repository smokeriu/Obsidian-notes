PyTorch可以通过`LSTM`函数定义LSTM层：
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
- `input_size`表示输入数据的大小。，则
- `hidden_size`表示每个隐藏层中，节点的个数。
- `num_layers`表示隐藏层的层数，默认为1。
- `bias`默认为`True`，为`False`时，则表示不使用偏置参数。
- `batch_first`默认为`False`，为`True`时表示输入张量第一维是批量大小。
	- 一般而言，循环神经网络的第一维表示序列长度，第二维。
- `dropout`默认为0，若为非0，则表示除了最后一层，其它层都会插入[暂退层](定义暂退层.md)。
- `bidirectional`默认为Flase，为`True`表示使用双向LSTM。
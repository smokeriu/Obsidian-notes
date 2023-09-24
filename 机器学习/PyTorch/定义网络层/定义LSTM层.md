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
- `num_layers`表示隐藏层的
用于定义[[../../深度学习/神经网络/现代神经循环网络/门控循环单元(GRU)|门控循环单元(GRU)]]。

```python
nn.GRU(
	input_size: int, 
	hidden_size: int, 
	num_layers: int,
	bias: bool = True,
	batch_first: bool = False,
	dropout: int = 0,
	bidirectional: bool = False
)
```

其中参数含义与[[定义LSTM层]]保持一致。
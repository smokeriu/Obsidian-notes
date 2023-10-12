seq2seq是[[编码器和解码器架构]]的一种具体实现，用于解决现实生活中，输入和输出句子长度不一致的问题。常用于翻译场景：
![](Pasted%20image%2020231010210622.png)

# 编码器

编码器用于将输入转换成state，这里使用的是[[../通用知识/嵌入层(Embedding)|嵌入层(Embedding)]]的机制来对数据进行编码，并使用[[门控循环单元(GRU)]]作为循环块：
```python
self.embedding = nn.Embedding(vocab_size, embed_size)
self.rnn = nn.GRU(embed_size, num_hiddens, num_layers,
	dropout=dropout)
```

在接收到Input数据X后，则会产生输出和state：
```python
X = self.embedding(X) # X是input
X = X.permute(1, 0, 2) # 因为GRU要求第一个维度为时间步，这里需要置换。
output, state = self.rnn(X)
```

这里：
- X最初的形状为：`(batch_size, num_steps)`。
	- 嵌入层后的形状为：`(batch_size, num_steps, embed_size)`。
- output的形状为：`(num_steps, batch_size, num_hiddens)`。
- state的形状为：`(num_layers, batch_size, num_hiddens)`。
> num_layers是隐藏层的层数；num_hiddens是单个隐藏层中，神经元的个数。

简单而言，state是GRU每一个隐藏层都会输出一个张量，其来自于最后一个时间步，相当于记录了这一层一整个时间步的内容。
# 解码器

# 损失函数
seq2seq是[[编码器和解码器架构]]的一种具体实现，用于解决现实生活中，输入和输出句子长度不一致的问题。常用于翻译场景：
![](Pasted%20image%2020231010210622.png)

# 编码器

编码器用于将输入转换成state，这里使用的是[[../通用知识/嵌入层(Embedding)|嵌入层(Embedding)]]的机制来对数据进行编码，并使用[[门控循环单元(GRU)]]作为循环块：
```python
self.embedding = nn.Embedding(vocab_size, embed_size)
self.rnn = nn.GRU(embed_size, num_hiddens, num_layers,
	dropout=dropout)
```

在接收到Input数据X后，则会产生state：
```python
X = self.embedding(X)
X = X.permute(1, 0, 2) # 因为GRU要求第一个维度为时间步，这里需要置换
```

# 解码器

# 损失函数
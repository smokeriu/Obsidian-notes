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

解码器与编码器的主要区别在于：
- 循环层输入时需要考虑编码器产出的状态。
- 需要一个线性层来输出。
	- 线性层的输出向量大小为vocab_size，即可以用于分类表示输出的字符是什么。
```python
self.embedding = nn.Embedding(vocab_size, embed_size)
self.rnn = nn.GRU(embed_size + num_hiddens, num_hiddens, num_layers,
	dropout=dropout)
self.dense = nn.Linear(num_hiddens, vocab_size)
```
另外，解码器还需要定义如何使用编码器产出的内容：
```python
def init_state(self, enc_outputs, *args):
	return enc_outputs[1]
```
> 这里的`enc_outputs`其实是一个二元组：(output, state)。

在前向传播时，解码器还需要根据state来定义上下文，即认为输入的句子的信息都已经存放在了state中：
```python
# arg: X, state
X = self.embedding(X).permute(1, 0, 2)
context = state[-1].repeat(X.shape[0], 1, 1)
X_and_context = torch.cat((X, context), 2)
output, state = self.rnn(X_and_context, state)
output = self.dense(output).permute(1, 0, 2)
```

这里：
- X最初的形状为：`(batch_size, num_steps2)`。
	- 这里用`num_steps2`主要是为了和编码器的`num_steps`区分开。
	- 嵌入层后的形状为：`(batch_size, num_steps2, embed_size)`。
- state的形状来源于编码器，为`(num_layers, batch_size, num_hiddens1)`。
	- `num_hiddens1`是编码器的每一个隐藏层的隐藏单元个数。
- context的形状由state变换得到，为`(1 * num_steps2, batch_size, num_hiddens1)`。
	- `state[-1]`表示只取编码器最后一个隐藏层的state。
	- 我们将这个状态拓展到与`num_steps2`一样的大小，即对于单个序列，其每个时间步看到的状态是一样的。
	- `X.shape[0]`中的X已经是转置后的。
- X_and_context的形状为：`(num_steps2, batch_size, embed_size + num_hiddens1)`。
	- 所以定义的GRU的`input_size`是`embed_size + num_hiddens`。
	- 可以认为，rnn的输入，实际是：解码器输入信息 + 编码器最后一个隐藏层的信息拼接得到的。
	- 则这里循环神经网络，实际上是将 翻译后的词元与翻译前句子的整体状态结合一起看。
- 循环层后output的形状为：`(num_steps2, batch_size, num_hiddens)`。
	- 线性后（转置前）的output的形状为：`(num_steps2, batch_size, vocab_size)`。

整体的架构如下：
![[assets/Pasted image 20231012113912.png|500]]

# 损失函数
在每个时间步，解码器预测了输出词元的概率分布。 类似于语言模型，可以使用softmax来获得分布， 并通过计算交叉熵损失函数来进行优化。

但这里还引入了填充次元，则在计算损失前，应当屏蔽不相关的项：
```python
def sequence_mask(X, valid_len, value=0):
	maxlen = X.size(1) # 得到输入的step数
	# mask是一个bool组成的张量。
	mask = torch.arange((maxlen), dtype=torch.float32,
                device=X.device)[None, :] < valid_len[:, None]
    # 这里对mask取反，将True对应的下标值设置的value
    X[~mask] = value
    return X
```
这里将不相关项替换为0，以便后面任何不相关预测的计算都是与零的乘积，结果都等于零。

这里：
- X是形状为`(batch_size, num_step, ...)`的输入。
- `[None, :]`和`[:, None]`的作用是将一维数据转换为二维。
特别的，这里的`X[~mask]`用于将超过`num_step`的数据置为空。

另外使用[[../../../PyTorch/损失函数/交叉熵|交叉熵]]作为损失函数：
```python
class MaskedSoftmaxCELoss(nn.CrossEntropyLoss):
	"""带遮蔽的softmax交叉熵损失函数"""
	# pred的形状：(batch_size,num_steps,vocab_size) # 预测结果
	# label的形状：(batch_size,num_steps) # 标签数据
	# valid_len的形状：(batch_size,) # 合法长度
	def forward(self, pred, label, valid_len):
		weights = torch.ones_like(label)
		# 得到每批次，每一时间步的权重
		weights = sequence_mask(weights, valid_len)
		self.reduction='none' # 计算损失时不合并
		# pred.permute(0, 2, 1) = (batch_size, vocab_size, num_steps)
		# label = (batch_size,num_steps)
		unweighted_loss = super(MaskedSoftmaxCELoss, self).forward(
			pred.permute(0, 2, 1), label)
		weighted_loss = (unweighted_loss * weights).mean(dim=1)

		return weighted_loss
```
这里对原有的交叉熵进行了一些修改，即通过`valid_len`和label来计算屏蔽了不相关项的损失值。


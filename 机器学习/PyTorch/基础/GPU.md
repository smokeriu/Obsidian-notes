我们一般希望通过GPU来训练模型，严格意义上，Pytorch使用`cuda`代表GPU，即NVIDIA的GPU核心。
对于多张GPU卡，则使用`torch.device(f'cuda:{i}')`来切换，`cuda:0`和`cuda`是等价的。
```python
torch.device('cpu'), torch.device('cuda'), torch.device('cuda:1')
```


我们一般希望通过GPU来训练模型，严格意义上，Pytorch使用`cuda`代表GPU，即NVIDIA的GPU核心。
对于多张GPU卡，则使用`torch.device(f'cuda:{i}')`来切换，`cuda:0`和`cuda`是等价的。
```python
torch.device('cpu'), torch.device('cuda'), torch.device('cuda:1')
```

我们可以将GPU检测放在一个方法中，这样就可以根据实际情况灵活切换设备。
```python
def try_gpu(i=0):  #@save
    """如果存在，则返回gpu(i)，否则返回cpu()"""
    if torch.cuda.device_count() >= i + 1:
        return torch.device(f'cuda:{i}')
    return torch.device('cpu')
```
其中，`i`代表第几块GPU，在家庭环境，一般就一台，所以默认值是0。当对应索引的GPU不存在时，会返回CPU。


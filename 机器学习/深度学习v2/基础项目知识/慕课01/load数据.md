在Torch中，一般通过实现`torch.utils.data.Dataset`的子类，并读取时调用`torch.utils.data.Dataloader`的实现。

# 构建Dataset

继承`torch.utils.data.Dataset`。

主要需要实现两个函数：
- `__getitemm__`：如何读取数据，参数是下标。
- `__len__`：返回数据集的大小。
```python
class BanknoteDataset(torch.utils.data.Dataset):  
    def __init__(self, data_path):  
        self.dataset = np.loadtxt(data_path, delimiter=',')  
  
    def __getitem__(self, idx):  
        item = self.dataset[idx]  
        x, y = item[:HP.in_features], item[HP.in_features:]  
        return (torch.Tensor(x).float().to(HP.device),  
                torch.Tensor(y).squeeze().long().to(HP.device))  
  
    def __len__(self):  
        return self.dataset.shape[0]
```

其中，这个样例是一个二分类问题，所以标签只有一个值，故在构建`y`时，通过[squeeze](维度变换.md#squeeze)来将维度压缩。

# 使用Dataloader

通过Dataloader可以实现数据的批量读取，并且可以定义一些其他的参数：
```python
dataset = BanknoteDataset(HP.devset_path)  
loader = DataLoader(dataset, 
					batch_size=HP.batch_size, shuffle=True, drop_last=True)
for batch in loader:
  pass
```

通过DataLoader，则一个循环中可以读取`HP.batch_size`条数据。其中batch的类型是List，元素取决于定义Dataset的`__getitem__`方法返回的内容。
对于本例，每个batch都是一个长度为2的List。第一个元素就是x，第二个元素就是y。不过是以批量的形式组成的Tensor，长度均为`HP.batch_size`。
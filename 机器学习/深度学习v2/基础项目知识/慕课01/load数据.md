在Torch中，一般通过实现`torch.utils.data.Dataset`的子类，并读取时调用`torch.utils.data.Dataloader`的实现。

# 构建Dataset
1. 继承`torch.utils.data.Dataset`。

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

# 使用Dataloader
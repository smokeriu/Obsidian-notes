GN可以看作是[[BatchNormalization]]和[[InstanceNormalization]]的结合，其差异如下图所示：
![[assets/Pasted image 20240116200159.png]]
所以可以将LN和IN看作是GN的极端表现。一般而言，我们要求Group的G取值是C的整数倍。

与其他归一化有所不同的是，在求均值前，我们需要先将数据Reshape到`[N, G，C//G , H, W]`，此时我们认为归一化纬度是`[C//G , H, W]`。

> 这里到`//`表示整除。一般是需要保证C能被G整除。

所以在处理前，我们先要对数据进行reshape（或者view）。使同一组的数据合并到一个纬度：
```python
N, C, H, W = x.shape
x = x.view(N, G, -1) # x.view(N, G, C//G, H, W)
```
后续的处理就与InstanceNormalization一致了。其实我们可以在reshape后，使用InstanceNormalization进行处理，只是需要将结果再reshape回去。

# PyTorch
PyTorch预置了GN，参考[[../../../PyTorch/定义网络层/定义Group归一化|定义Group归一化]]。

# Why



# 参考
- [ECCV 2018 Open Access Repository](https://openaccess.thecvf.com/content_ECCV_2018/html/Yuxin_Wu_Group_Normalization_ECCV_2018_paper.html)
- 
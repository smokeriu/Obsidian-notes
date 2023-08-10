用于将两个张量拼接在一起，和Pandas的[[../../Pandas/基础/连接数据#concat|连接数据]]类似，也分为横向或纵向拼接。

```python
torch.cat(tensors: Union[Tuple[Tensor, ...], List[Tensor]], 
		  dim: _int=0)
```
其中：
- `tensors`表示需要关联的张量，一般为一个列表。
- `dim`表示关联维度。
	- 默认为0，表示竖向拼接，对矩阵的表现类似于`union`。

需要注意，因为张量可以是N维的，所以dim也可以是`N-1`的数字，则表示按照第几维拼接。

例如，我们
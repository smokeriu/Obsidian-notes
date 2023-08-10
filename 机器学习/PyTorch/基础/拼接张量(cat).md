用于将两个张量拼接在一起，和Pandas的[[../../Pandas/基础/连接数据#concat|连接数据]]类似，也分为横向或纵向拼接。

```python
torch.cat(tensors: Union[Tuple[Tensor, ...], List[Tensor]], 
		  dim: _int=0)
```
其中：
- `tensors`表示需要关联的张量，一般为一个列表。
- dim表示关联维度。
	- mo'rf
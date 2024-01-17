GN可以看作是[[BatchNormalization]]和[[InstanceNormalization]]的结合，其差异如下图所示：
![[assets/Pasted image 20240116200159.png]]
所以可以将LN和IN看作是GN的极端表现。一般而言，我们要求Group的G取值是C的整数倍。

与其他归一化有所不同的是，在求均值前，我们需要先将数据Reshape到`[N, G，C//G , H, W]`，此时我们认为归一化纬度是`[C//G , H, W]`。

> 这里到`//`表示整除。一般是需要保证C能被G整除。

所以在处理前，我们先要对数据进行reshape（或者view）。使同一组的数据合并到一个纬度：
``

其实现代码如下：
```python
def GroupNorm(x, gamma, beta, G, eps=1e-5):
    # x: input features with shape [N,C,H,W]
    # gamma, beta: scale and offset, with shape [1,C,1,1]
    # G: number of groups for GN
    N, C, H, W = x.shape
    x = tf.reshape(x, [N, G, C // G, H, W])
    mean, var = tf.nn.moments(x, [2, 3, 4], keep dims=True)
    x = (x - mean) / tf.sqrt(var + eps)
    x = tf.reshape(x, [N, C, H, W])
    return x * gamma + beta
```
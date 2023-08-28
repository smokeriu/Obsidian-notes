> 是[[残差网络(ResNet)]]和[[../../../../数学/基础/微积分/泰勒公式|泰勒公式]]的结合。

在ResNet中，我们函数$f$拆分为两部分：一个简单的线性项$x$和一个复杂的非线性项$g(x)$。不过二者是简单相加的。
DenseNet中，我们通过*连接*来替代简单相加：
$$
\mathbf{x} \to \left[
\mathbf{x},
f_1(\mathbf{x}),
f_2([\mathbf{x}, f_1(\mathbf{x})]), f_3([\mathbf{x}, f_1(\mathbf{x}), f_2([\mathbf{x}, f_1(\mathbf{x})])]), \ldots\right].
$$
即他们的关系变为了：
![[assets/Pasted image 20230828164704.png|500]]
> 上图中，左侧围ResNet，右侧为DenseNet。

整体上，DenseNet由*稠密块*(Dense block)和*过渡层*(transition layer)组成。前者定义如何连接输入和输出，而后者则控制通道数量，使其不会太复杂。

# 稠密块
稠密块即上文所说的，由多个卷积块组成，每个卷积块使用*相同数量的输出通道*。在前向传播中，我们将每个卷积块的输入和输出在**通道维**上连结。

显然，随着稠密块的加深，输出通道呈现明显的增长。
# 过渡层
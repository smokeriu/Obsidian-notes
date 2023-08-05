> Softmax回归其实是一个分类问题。

# Softmax定义
Softmax是一个函数，其作用于输出上。对于分类问题，网络会产生n个输出，组成一个n维向量，向量中的每个值表示预测结果**属于这个分类的置信度**。

Softmax作用于这个输出向量$\mathbf{o}$上，得到一个预测向量$\hat{\mathbf{y}}$：
$$
\hat{\mathbf{y}} = softmax(\mathbf{o})
$$
而具体的Softmax函数，则作用于向量的每个元素上，可以写作：
$$
\hat{y_i} = \frac{e^{o_i}}{\sum_k e^{o_k}}
$$
相当于将预测结果作为自然指数，


# Softmax与交叉熵
Softmax一般会和[交叉熵代价函数](交叉熵代价函数.md)一起使用，即我们可以得到代价函数：
$$
l(\mathbf{y}, \hat{\mathbf{y}}) = 
- \sum_i y_i log{\hat{y_i}}
$$
其中：
- $\mathbf{y}$是目标值，其是一个n维向量，且只有第$i$位为1，其余位为0。表示结果为第$i$位表示的分类项。
- $\hat{\mathbf{y}}$是一个预测值，其是作用于预测输出的Softmax函数的结果。
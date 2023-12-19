是各种缓冲区的组合，就是把他们封装到一个缓冲区，或者合并成一个缓冲区。最常见的比如`HTTP`的一些头信息封装在一个缓冲区，消息体在另一个。或者你自己定义一个协议，比如可以分成好几个缓冲区。

# 结构
其结构如图：
![[assets/Pasted image 20231219100241.png]]
里面有个`Component`数组，`Component`里面放着缓冲区，还有各种索引。

## Component
属性：
- `offset`和`endOffset`：表示当前缓冲区相对于整个`CompositeByteBuf`可以读取的范围，比如说有三个缓冲区，每个缓冲区只写了`10`个字节，那第一个缓冲区的索引就是`0-9`，第二个是`10-19`，第三个是`20-29`
- `srcAdjustment`和`adjustment`：起始索引的*读索引*偏移，大多数是负的，因为这样直接获取`CompositeByteBuf`索引的值的时候，可以直接定位到`buf`里的读索引位置。

## 主要方法


![[assets/Pasted image 20231219100921.png]]

# 参考：
- [吃透Netty源码系列三十五之CompositeByteBuf详解一-CSDN博客](https://blog.csdn.net/wangwei19871103/article/details/104486129)